---
title: elf格式要求
date: 2022-10-21 20:41:12
categories: Loader
tags: [loader,elf]
---
由于经过bolt处理后的二进制文件，其codesize有膨胀。于是，针对codesize，对bolt做了一些优化。过程中遇到了一些问题，经调试动态连接器，了解了一些elf文件的格式要求。
## segment p_vaddr与p_offset
- 类型为Load的segment，p_vaddr与p_offset要关于segment align 同余。
- 类型为Load的segment，务必以page size对齐
```c++
dl-load.c

struct link_map *
_dl_map_object_from_fd (const char *name, const char *origname, int fd,
			struct filebuf *fbp, char *realname,
			struct link_map *loader, int l_type, int mode,
			void **stack_endp, Lmid_t nsid)
{
    ...
      {
    /* Scan the program header table, collecting its load commands.  */
    struct loadcmd loadcmds[l->l_phnum];
    size_t nloadcmds = 0;
    bool has_holes = false;

    /* The struct is initialized to zero so this is not necessary:
    l->l_ld = 0;
    l->l_phdr = 0;
    l->l_addr = 0; */
    for (ph = phdr; ph < &phdr[l->l_phnum]; ++ph)
      switch (ph->p_type)
	{
    ...
	case PT_LOAD:
	  /* A load command tells us to map in part of the file.
	     We record the load commands and process them all later.  */
	  if (__glibc_unlikely ((ph->p_align & (GLRO(dl_pagesize) - 1)) != 0))
	    {
	      errstring = N_("ELF load command alignment not page-aligned");
	      goto call_lose;
	    }
	  if (__glibc_unlikely (((ph->p_vaddr - ph->p_offset)
				 & (ph->p_align - 1)) != 0))
	    {
	      errstring
		= N_("ELF load command address/offset not properly aligned");
	      goto call_lose;
	    }
        ...
    }
    ...
}
```
## Load segment的排列顺序
类型为DYN的可执行程序或动态库，Load segment要按照地址从小到大布局
```c++
struct link_map *
_dl_map_object_from_fd (const char *name, const char *origname, int fd,
			struct filebuf *fbp, char *realname,
			struct link_map *loader, int l_type, int mode,
			void **stack_endp, Lmid_t nsid)
{
    ...
  {
    /* Scan the program header table, collecting its load commands.  */
    struct loadcmd loadcmds[l->l_phnum];
    size_t nloadcmds = 0;
    bool has_holes = false;

    /* The struct is initialized to zero so this is not necessary:
    l->l_ld = 0;
    l->l_phdr = 0;
    l->l_addr = 0; */
    for (ph = phdr; ph < &phdr[l->l_phnum]; ++ph)
      switch (ph->p_type)
	{
	  ...
	case PT_LOAD:
	  /* A load command tells us to map in part of the file.
	     We record the load commands and process them all later.  */
	  if (__glibc_unlikely ((ph->p_align & (GLRO(dl_pagesize) - 1)) != 0))
	    {
	      errstring = N_("ELF load command alignment not page-aligned");
	      goto call_lose;
	    }
	  if (__glibc_unlikely (((ph->p_vaddr - ph->p_offset)
				 & (ph->p_align - 1)) != 0))
	    {
	      errstring
		= N_("ELF load command address/offset not properly aligned");
	      goto call_lose;
	    }

	  struct loadcmd *c = &loadcmds[nloadcmds++];
	  c->mapstart = ALIGN_DOWN (ph->p_vaddr, GLRO(dl_pagesize));
	  c->mapend = ALIGN_UP (ph->p_vaddr + ph->p_filesz, GLRO(dl_pagesize));
	  c->dataend = ph->p_vaddr + ph->p_filesz;
	  c->allocend = ph->p_vaddr + ph->p_memsz;
	  c->mapoff = ALIGN_DOWN (ph->p_offset, GLRO(dl_pagesize));

	  /* Determine whether there is a gap between the last segment
	     and this one.  */
	  if (nloadcmds > 1 && c[-1].mapend != c->mapstart)
	    has_holes = true;

	  /* Optimize a common case.  */
#if (PF_R | PF_W | PF_X) == 7 && (PROT_READ | PROT_WRITE | PROT_EXEC) == 7
	  c->prot = (PF_TO_PROT
		     >> ((ph->p_flags & (PF_R | PF_W | PF_X)) * 4)) & 0xf;
#else
	  c->prot = 0;
	  if (ph->p_flags & PF_R)
	    c->prot |= PROT_READ;
	  if (ph->p_flags & PF_W)
	    c->prot |= PROT_WRITE;
	  if (ph->p_flags & PF_X)
	    c->prot |= PROT_EXEC;
#endif
	  break;
    ...

    /* Length of the sections to be loaded.  */
    maplength = loadcmds[nloadcmds - 1].allocend - loadcmds[0].mapstart;

    /* Now process the load commands and map segments into memory.
       This is responsible for filling in:
       l_map_start, l_map_end, l_addr, l_contiguous, l_text_end, l_phdr
     */
    errstring = _dl_map_segments (l, fd, header, type, loadcmds, nloadcmds,
				  maplength, has_holes, loader);
    if (__glibc_unlikely (errstring != NULL))
      goto call_lose;
  }
}
```
若不按地址大小排列，则maplength可能为负值。_dl_map_segments 函数如下：
```c++
elf/dl-map-segments.h

static __always_inline const char *
_dl_map_segments (struct link_map *l, int fd,
                  const ElfW(Ehdr) *header, int type,
                  const struct loadcmd loadcmds[], size_t nloadcmds,
                  const size_t maplength, bool has_holes,
                  struct link_map *loader)
{
  const struct loadcmd *c = loadcmds;

  if (__glibc_likely (type == ET_DYN))
    {
      /* This is a position-independent shared object.  We can let the
         kernel map it anywhere it likes, but we must have space for all
         the segments in their specified positions relative to the first.
         So we map the first segment without MAP_FIXED, but with its
         extent increased to cover all the segments.  Then we remove
         access from excess portion, and there is known sufficient space
         there to remap from the later segments.

         As a refinement, sometimes we have an address that we would
         prefer to map such objects at; but this is only a preference,
         the OS can do whatever it likes. */
      ElfW(Addr) mappref
        = (ELF_PREFERRED_ADDRESS (loader, maplength,
                                  c->mapstart & GLRO(dl_use_load_bias))
           - MAP_BASE_ADDR (l));

      /* Remember which part of the address space this object uses.  */
      l->l_map_start = (ElfW(Addr)) __mmap ((void *) mappref, maplength,
                                            c->prot,
                                            MAP_COPY|MAP_FILE,
                                            fd, c->mapoff);
      if (__glibc_unlikely ((void *) l->l_map_start == MAP_FAILED))
        return DL_MAP_SEGMENTS_ERROR_MAP_SEGMENT;

      l->l_map_end = l->l_map_start + maplength;
      l->l_addr = l->l_map_start - c->mapstart;

      if (has_holes)
        {
          /* Change protection on the excess portion to disallow all access;
             the portions we do not remap later will be inaccessible as if
             unallocated.  Then jump into the normal segment-mapping loop to
             handle the portion of the segment past the end of the file
             mapping.  */
          if (__glibc_unlikely
              (__mprotect ((caddr_t) (l->l_addr + c->mapend),
                           loadcmds[nloadcmds - 1].mapstart - c->mapend,
                           PROT_NONE) < 0))
            return DL_MAP_SEGMENTS_ERROR_MPROTECT;
        }

      l->l_contiguous = 1;

      goto postmap;
    }

  /* Remember which part of the address space this object uses.  */
  l->l_map_start = c->mapstart + l->l_addr;
  l->l_map_end = l->l_map_start + maplength;
  l->l_contiguous = !has_holes;

  while (c < &loadcmds[nloadcmds])
    {
      if (c->mapend > c->mapstart
          /* Map the segment contents from the file.  */
          && (__mmap ((void *) (l->l_addr + c->mapstart),
                      c->mapend - c->mapstart, c->prot,
                      MAP_FIXED|MAP_COPY|MAP_FILE,
                      fd, c->mapoff)
              == MAP_FAILED))
        return DL_MAP_SEGMENTS_ERROR_MAP_SEGMENT;

    postmap:
    ...

  /* Notify ELF_PREFERRED_ADDRESS that we have to load this one
     fixed.  */
  ELF_FIXED_ADDRESS (loader, c->mapstart);

  return NULL;
}
```
这里可以看到，_dl_map_segments处理类型为DYN和EXEC的elf文件方式不同。
- type为DYN，直接用maplength做内存映射，因此maplength务必是load segments的有效总大小
- type为EXEC，分别对每个segment做内存映射

综上，类型为DYN的elf文件，segment table中，Load segment务必按地址从小到大排列。另外需要注意的是，在编译可执行文件时，如果加了pie选项，则该生成的可执行文件类型为DYN，同样要遵循上述规则。