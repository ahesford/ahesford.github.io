# About

Twenty years of experience in applying computational science, numerical
analysis and scientific computing to solve challenging problems in the areas of
medical imaging, organizational behavioral modeling, radio antenna and
high-speed electronic circuit design, video and image processing, financial
analysis, monitoring of manufacturing processes, and performance testing.

## What I Do

I design and implement novel tools to route and analyze data, report results,
synthesize models, simulate outcomes and make decisions. Parallel algorithms
developed by me solve numerical problems of record-breaking scale. Live
exploration is a key aspect of the tools I build, supplementing automated
analysis and reporting with the ability to transform and review calculations on
the fly. I am keenly interested in broadening the applications of numerical and
scientific computing into nontraditional applications and offering unique
insight into a wide range of problems.

## Public Projects

Although much of my professional effort is devoted to proprietary projects,
some of my past work and current diversions are available in public software
repositories. Many of these are academic projects and, because they were
intended to support my own research, are not thoroughly documented.

- [pycwp](https://github.com/ahesford/pycwp), a Python package that provides
  routines for computational wave physics, with an emphasis on acoustic waves.

- [habis-tools](https://github.com/ahesford/habis-tools), a Python package that
  facilitates medical imaging using measurements of acoustic scattering
  collected by a novel imaging system developed at the University of Rochester.

- [fastsphere](https://github.com/ahesford/fastsphere), a C program designed to
  simulate the acoustic scattering response of a collection of spherical
  objects with differing properties.

- [afma](https://github.com/ahesford/afma), C programs for forward and inverse
  acoustic scattering by arbitrary, inhomogeneous, three-dimensional media;
  uses MPI and OpenMP to support massively parallel distributed computing
  systems. This program relies on the ScaleME fast multipole library, which is
  a project of the University of Illinois.

- [gpg_unlock](https://github.com/ahesford/gpg_unlock), a Python utility used
  by the `pam_exec.so` PAM module to unlock GPG keys on user login.

- [duiadns](https://github.com/ahesford/duiadns), a Python program to register
  dynamic host records for the [DUIS DNS service](https://www.duiadns.net).

- [bonjour-repeater](https://github.com/ahesford/bonjour-repeater), a Python
  utility that repeats mDNS (Bonjour) records with optional transformations.
  This program was written to convert CUPS shared printer records into a format
  recognized by iOS as AirPrint printers.

## Diversions

In my free time, I contribute to a some open-source projects. My
[GitHub profile](https://github.com/ahesford) contains forks of some projects I
contribute to or find interesting, in addition to those described above.

If you are looking for an operating system, I highly recommend
[Void Linux](https://www.voidlinux.org).

When I build Linux systems, I rely on [OpenZFS](https://github.com/openzfs/zfs)
for storage because of its excellent and flexible volume management and
replication features. The best way to boot a Linux system from ZFS is
[zfsbootmenu](https://github.com/zdykstra/zfsbootmenu), a project on which I
collaborate to bring FreeBSD-style ZFS boot environments to Linux.

## Articles

I've written a few articles that describe my Void Linux setup or its use. They
are reproduced here for general interest.

* Using ZFS snapshots and `zfsbootmenu` to bisect a `libvirt` "regression";
  [local copy](./articles/libvirt-zbm-notes.html) or
  [Reddit original](https://www.reddit.com/r/voidlinux/comments/hmnzxt/using_zfs_snapshots_and_zfsbootmenu_to_bisect_a/)

* Simple backups with `zfs-auto-snapshot`, `zfs-prune-snapshots` and `zrep`;
  [local copy](./articles/zfs-backup-strategies.html) or
  [Reddit original](https://www.reddit.com/r/voidlinux/comments/hu1ron/simple_backup_with_zfsautosnapshot/)

# Contact

Find me on [LinkedIn](https://www.linkedin.com/in/ajhesford/) for more
information about my work.

For a more immediate response, find me as `ahesford` on the `#voidlinux`
[freenode](https://freenode.net) IRC channel.
