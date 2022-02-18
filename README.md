# docker-on-tmpfs
GitHub Action which mounts a tmpfs volume on `/var/lib/docker`.

This action was created to solve a [very specific problem][1] that occurs when
building the latest pip `cryptography` package inside a QEMU emulated
`linux/arm/v7` environment (e.g. Rasberry Pi) on a `linux/amd64` computer.

The compilation of the package fails because some weirdness inside a low level
library when it tries to read the filesytem while running inside a 32-bit
environment that is emulated by QEMU running on a 64-bit host. Apparently a
workaround for this is to swap the filesystem to a tmpfs one in order to not
trigger this bug, so we make the entire `/var/lib/docker` folder like that.

Here are links to the biggest threads discussing this issue further:

- https://github.com/docker/buildx/issues/395
- https://github.com/rust-lang/cargo/issues/8719
- https://gitlab.com/qemu-project/qemu/-/issues/263


### Acknowledgments and Thanks

This action has been inspired by the workarounds posted in the threads mentioned
above, but there are three solutions I would like to give extra credit to:

- [`@pierotofy`][4]: For showing how to configure the swap space.
- [`@easimon`][5]: For providing details about GitHub runners.
- [`@nijel`][6]: For showing how a `tmpfs` can be used as a workaround.


## Usage

> :warning: Please read the [More Information](#more-information) section to
            understand the physical limitations of the GitHub runners before
            changing the values.

```yaml
- name: Run Docker on tmpfs
  uses: JonasAlfredsson/docker-on-tmpfs@v1.0.0
  with:
    tmpfs_size: 5
    swap_size: 4
    swap_location: '/mnt/swapfile'
```


## More Information

The official [specifications][2] of GitHub runner machines are the following:

- 2-core CPU
- 7 GB of RAM memory
- 14 GB of SSD disk space

However, this does not paint the whole picture, and a more detailed
representation of a newly starter runner would be this:

#### CPU
```
vendor_id	: GenuineIntel
cpu family	: 6
model		: 85
model name	: Intel(R) Xeon(R) Platinum 8171M CPU @ 2.60GHz
cpu MHz		: 2095.197
cache size	: 36608 KB
flags		: fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ss ht syscall nx pdpe1gb rdtscp lm constant_tsc rep_good nopl xtopology cpuid pni pclmulqdq ssse3 fma cx16 pcid sse4_1 sse4_2 movbe popcnt aes xsave avx f16c rdrand hypervisor lahf_lm abm 3dnowprefetch invpcid_single pti fsgsbase bmi1 hle avx2 smep bmi2 erms invpcid rtm mpx avx512f avx512dq rdseed adx smap clflushopt avx512cd avx512bw avx512vl xsaveopt xsavec xsaves md_clear
bugs		: cpu_meltdown spectre_v1 spectre_v2 spec_store_bypass l1tf mds swapgs taa itlb_multihit
```

#### Memory

```
              total        used        free      shared  buff/cache   available
Mem:          6.8Gi       484Mi       5.3Gi       9.0Mi       1.0Gi       6.0Gi
Swap:         4.0Gi          0B       4.0Gi
```

```
NAME          TYPE SIZE USED PRIO
/mnt/swapfile file   4G   0B   -2
```

#### Storage

```
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda       8:0    0   14G  0 disk
└─sda1    8:1    0   14G  0 part /mnt
sdb       8:16   0   86G  0 disk
├─sdb1    8:17   0 85.9G  0 part /
├─sdb14   8:30   0    4M  0 part
└─sdb15   8:31   0  106M  0 part /boot/efi
```

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/root        84G   53G   31G  63% /
devtmpfs        3.4G     0  3.4G   0% /dev
tmpfs           3.4G  4.0K  3.4G   1% /dev/shm
tmpfs           695M  1.1M  694M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           3.4G     0  3.4G   0% /sys/fs/cgroup
/dev/sdb15      105M  5.2M  100M   5% /boot/efi
/dev/sda1        14G  4.1G  9.0G  32% /mnt
```

#### Default Workspace

There is one variable that is extra interesting of those listed [here][3], and
that is `GITHUB_WORKSPACE` which is explained like this:

> Actions and shell commands execute in this directory. An action can modify
  the contents of this directory, which subsequent actions can access.

This path defaults to

    /home/runner/work/<repo-name>/<repo-name>

so in this repository's case it would be

    /home/runner/work/docker-on-tmpfs/docker-on-tmpfs

This is also the path you will get if you run just `echo ${PWD}`.

### Limitations

With the above information we can see that, by default, we are not using the
advertised 14G available on `/mnt`, but instead the 31G on `/`. Furthermore,
of those 14G on `/mnt` 4 are used for the swap file, so it is really only 10G
available there. At startup we are also using ~0.5G RAM, which means that we
have about 6G to play with for our `tmpfs` volume.

What is important to understand is that a `tmpfs` volume writes its data to RAM,
so if you consume more than the available the system will start moving data to
the swap file on the disk. Reading data from disk is **brutally** slow compared
to accessing it from RAM, so when this happens the system might become unusable
slow. However, if you remove the swap file and fill up the RAM the system will
probably crash so it is important to keep if we are to do what this action does.
Linux is also pretty clever, so data more frequently accessed is less likely to
be pushed to swap.

I would suggest you try to keep `tmpfs_size` as small as possible for your
task, in order to minimize the risk of starving the other system processes.
A good thing to do is also to keep the `swap_size` the same size as your
`tmpfs`, and this is even more important in case you want to make the volume
larger than the currently available RAM.






[1]: https://github.com/JonasAlfredsson/docker-nginx-certbot/issues/30
[2]: https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners
[3]: https://docs.github.com/en/actions/learn-github-actions/environment-variables
[4]: https://github.com/pierotofy/set-swap-space
[5]: https://github.com/easimon/maximize-build-space
[6]: https://github.com/WeblateOrg/docker/pull/1274/commits/c40a9949596cee31d6a56597e5e3480e0b090d25
