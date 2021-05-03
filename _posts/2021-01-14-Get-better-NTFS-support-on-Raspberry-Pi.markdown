---
layout: post
title:  "Get better NTFS support on Raspberry Pi"
date:   2021-01-14 00:00:00 +0000
categories: 
excerpt: "We all know how Raspberry Pi is such a great little device. Numerous use cases have been found for it, including being used as a file server. Personally, the one I have at home, besides doing some other tasks, also takes care of exposing a USB SSD using Samba on the network, where my Windows PC regularly saves backups using File History. A new NTFS driver is about to be upstreamed that provides very good performance, so today, we are attempting to install it on the slightly older kernel the Pi4 ships with."
---

Update (May 3rd, 2021): I have updated this post with a newer installation script that patches the latest version of the driver available on the mailing list (v26) to work with the 5.4 kernel on Raspberry Pi.

We all know how Raspberry Pi is such a great little device. Numerous use cases have been found for it, including being used as a file server. Personally, the one I have at home, besides doing some other tasks, also takes care of exposing a USB SSD using [Samba](https://www.samba.org/) on the network, where my Windows PC regularly saves backups using [File History](https://support.microsoft.com/en-us/windows/file-history-in-windows-5de0e203-ebae-05ab-db85-d5aa0a199255).

One of the problems for such a setup is what file system to choose for the external hard drive where you store the backups. Since I use Windows on some computers, it's been proven on a number occasions that it is very convenient that, in case of an emergency, one can simply plug the drive in a Windows machine and explore its contents freely using File Explorer. For this to work, the drive's file system has to be supported in Windows in one form or another.

I have used a number of strategies in the past:

* Format the drive as ext4, which offers the best performance in Linux, since it is natively supported, and then use some third party driver in Windows, like [Ext2Fsd](http://www.ext2fsd.com/); the problem with this solution is that you have to rely on some third party component that is not part of Windows, sometimes it may be closed source, and it is prone to being left behind and becoming outdated (that's what happened largely with Ext2Fsd, but it seems [someone has picked up development again](https://github.com/bobranten/Ext4Fsd)).
* Format the drive as ext4 and use Windows Subsystem for Linux. Specifically, with WSL2 and some recent Windows 10 builds, Microsoft allows mounting physical disks directly in the Linux subsystem, as explained [here](https://docs.microsoft.com/en-us/windows/wsl/wsl2-mount-disk); the problem is, it is not available in the latest stable Windows 10 build (20h2, 19042), plus, you still rely on WSL2, so another abstraction etc - kind of hard to use in an emergency.
* Format the drive as FAT32 - not a solution because I have files larger than 4GB.
* Format the drive as exFAT. The problem with this is that exFAT suffers from fragmentation problems, plus not the best support on power losses etc. It really is more of a file system designed for thumb drives (i.e. removable media), not disks running 24/7. There is a [FUSE](https://en.wikipedia.org/wiki/Filesystem_in_Userspace) driver implementation that ships with the latest Raspbian, plus newer Linux kernels have a built-in module.
* Format the drive as NTFS; this time, we need a solution on Linux. The kernel still ships with a very outdated NTFS driver that allows browsing drives only read only. Fortunately, most distros include the [ntfs-3g](https://www.tuxera.com/company/open-source/) package, a FUSE driver which is pretty good.

In the end, I settled with the last solution, as a good balance between convenience and performance. On Raspberry Pi, when mounting a drive using ntfs-3g, make sure to specify the option `big_writes` in order to have acceptable performance, like so:

```
# mount -o big_writes /dev/sdb1 /mnt
```

The problem with ntfs-3g is that it is kind of slow. Even with `big_writes`, it simply cannot saturate the Gigabit link between the Pi and the computer. That would mean around 125 MB/s. That's the speed I get when transferring over Samba to the SSD I boot the Pi from which is ext4 formatted. With ntfs-3g, it fluctuates between 55 MB/s and the best I could get is around 90 MB/s. So quite a drop in performance. The CPU is almost maxed out - a lot of context switches happen, and it all boils down to, probably, the fact that we are talking about a user space implementation, which simply has these kind of limitations.

Now, 80% performance in the best scenarios is not that bad, but, I don't know, it is 2021, not even saturating a gigabit connection is pretty lame. I was about to format the external drive as exFAT and do some tests, then decided to give up on that and go with good old trusted ext4, when I accidentally learned about a new driver that Paragon Software is contributing (these guys do all kinds of file systems utilities for Windows/macOS/Linux) to the kernel. And it is not only talks, they have actually submitted code for review, the most recent patch a few days ago. Read [here](https://www.phoronix.com/scan.php?page=news_item&px=NTFS-Linux-Driver-V16), for example.

Now, that's what I call news! That's awesome. I naturally wanted to use that. Knowing that it will eventually be upstreamed and included with the kernel definitely makes it a very good candidate. Sure, it is going to take a while before it officially reaches the Pi, but it is nevertheless promising.

Since I don't want to mess with any `rpi-update` and upgrade the kernel of my working setup, I decided to try it on the current kernel and patch my way if the situation demands it. So, let's compile and install that on the Raspberry Pi.

The first step when you want to do such things is to look on the [Arch Wiki](https://wiki.archlinux.org/) or in the [Arch User Repository](https://aur.archlinux.org/) for any relevant packages regarding this. Arch is a very good distro, very well documented, it is my go to choice when I have the opportunity. On Raspberry Pi, of course, I run Raspbian since Arch is not really an option. It does not matter that Raspbian is Debian while Arch is Arch - they use a very friendly packaging system and we can get a lot of useful info from there and come up with a script that will run just fine on the Pi.

A good candidate for our search is [ntfs3-dkms](https://aur.archlinux.org/packages/ntfs3-dkms/) from AUR. Looking at the PKGBUILD, I can see it downloads the files directly off the mailing list in the form of patches which apply one after the other to generate the files for the driver. What's great about this package is that it comes with a proper dkms.conf file, so it is really easy to integrate this with DKMS, which will make sure the driver gets rebuilt when the kernel is updated etc. Also, another advantage is that is gets properly installed in the system, so it can be loaded automatically on demand, without you having to do any `insmod ntfs3.ko` beforehand.

So, I came up with an installation script that largely mimics the PKGBUILD of the Arch package. The single issue I have found is that the driver does not compile as is on Raspberry Pi. The kernel on my Pi is `5.4.72-v7l+` (`uname -r`). This kernel does not have support for the [readahead](https://elixir.bootlin.com/linux/latest/C/ident/readahead_control) operation in file systems. This has been introduced around version 5.8 as far as I can tell. Fortunately, this is not that big of a thing - it is just an improvement that the kernel offers that drivers can take advantage of. This new driver takes advantage of that, but since our current kernel does not support that, we can take out that support (which equates to deleting a few lines from a certain source file) and then it compiles and works just fine. 

Update (May 3rd, 2021): I have updated the script to work with the latest v26 of the driver, the very latest submitted for review. For this to work with kernel 5.4, a few more patches are needed, namely:

1) *readahead* support in *struct address_space_operation* (this was already a problem with v17 for which I made a patch)

2) BIO_MAX_VECS is called BIO_MAX_PAGES in kernel 5.4

3) callback prototypes in *struct inode_operations* do not take a *struct user_namespace** parameter as first argument in 5.4; this is due to a very recent change in the kernel (it made it into 5.12), where it was required to add this information to these callbacks; more details [here](https://lists.linuxfoundation.org/pipermail/containers/2020-October/042517.html)

For (1), we already have a patch from the previous version. For (2), it is easy to replace BIO_MAX_VECS with BIO_MAX_PAGE. For (3), my approach is to patch out all functions in the driver that take a *struct user_namespace** and remove that. For the functions where such a pointer is still required (like *posix_acl_from_xattr*), the fallback variable to provide as an argument is *init_user_ns* (the driver already does this in a few places - that's the "initial user  namespace" which makes everything behave like in older kernels, so to  say).

Note that (3) is required even on the newer 5.11 kernel in more recent Raspbian releases, as the *struct inode_operations* was changed only in 5.12.

I have included the relevant patch in the installation script (make sure to run it as root, so with `sudo`):

{% highlight bash%}
#!/bin/bash
pkgname=ntfs3
pkgver=26.0.0
prefix=/usr/src

mkdir -p ${prefix}/${pkgname}-${pkgver}
cd ${prefix}/${pkgname}-${pkgver}
for i in `seq 2 9`; do
	wget -O p$i https://lore.kernel.org/lkml/20210402155347.64594-$i-almaz.alexandrovich@paragon-software.com/raw
	patch -p3 -N -i p$i
	rm p$i
done

wget -O Makefile.patch https://aur.archlinux.org/cgit/aur.git/plain/Makefile.patch?h=ntfs3-dkms
patch -p0 -N -i "Makefile.patch"
rm Makefile.patch

patch --ignore-whitespace Makefile << 'EOT'
--- a/Makefile
+++ b/Makefile
@@ -37,7 +37,7 @@ ccflags-$(CONFIG_NTFS3_LZX_XPRESS) += -DCONFIG_NTFS3_LZX_XPRESS
 ccflags-$(CONFIG_NTFS3_FS_POSIX_ACL) += -DCONFIG_NTFS3_FS_POSIX_ACL
 
 all:
-	make -C /lib/modules/$(KVERSION)/build M=$(PWD) modules
+	make -C /lib/modules/$(KVERSION)/build M=$(shell pwd) modules
 
 clean:
-	make -C /lib/modules/$(KVERSION)/build M=$(PWD) clean
\ No newline at end of file
+	make -C /lib/modules/$(KVERSION)/build M=$(shell pwd) clean

EOT

patch --ignore-whitespace fsntfs.c << EOT
--- a/fsntfs.c
+++ b/fsntfs.c
@@ -1620,7 +1620,7 @@ int ntfs_bio_fill_1(struct ntfs_sb_info *sbi, const struct runs_tree *run)
 		lbo = (u64)lcn << cluster_bits;
 		len = (u64)clen << cluster_bits;
 new_bio:
-		new = ntfs_alloc_bio(BIO_MAX_VECS);
+		new = ntfs_alloc_bio(BIO_MAX_PAGES);
 		if (!new) {
 			err = -ENOMEM;
 			break;
EOT

patch --ignore-whitespace ntfs_fs.h << 'EOT'
--- a/ntfs_fs.h
+++ b/ntfs_fs.h
@@ -453,11 +453,11 @@ bool dir_is_empty(struct inode *dir);
 extern const struct file_operations ntfs_dir_operations;
 
 /* globals from file.c*/
-int ntfs_getattr(struct user_namespace *mnt_userns, const struct path *path,
+int ntfs_getattr(const struct path *path,
 		 struct kstat *stat, u32 request_mask, u32 flags);
 void ntfs_sparse_cluster(struct inode *inode, struct page *page0, CLST vcn,
 			 CLST len);
-int ntfs3_setattr(struct user_namespace *mnt_userns, struct dentry *dentry,
+int ntfs3_setattr(struct dentry *dentry,
 		  struct iattr *attr);
 int ntfs_file_open(struct inode *inode, struct file *file);
 int ntfs_fiemap(struct inode *inode, struct fiemap_extent_info *fieinfo,
@@ -644,7 +644,7 @@ int ntfs_sync_inode(struct inode *inode);
 int ntfs_flush_inodes(struct super_block *sb, struct inode *i1,
 		      struct inode *i2);
 int inode_write_data(struct inode *inode, const void *data, size_t bytes);
-struct inode *ntfs_create_inode(struct user_namespace *mnt_userns,
+struct inode *ntfs_create_inode(
 				struct inode *dir, struct dentry *dentry,
 				const struct cpu_str *uni, umode_t mode,
 				dev_t dev, const char *symname, u32 size,
@@ -784,17 +784,17 @@ int ntfs_cmp_names_cpu(const struct cpu_str *uni1, const struct le_str *uni2,
 /* globals from xattr.c */
 #ifdef CONFIG_NTFS3_FS_POSIX_ACL
 struct posix_acl *ntfs_get_acl(struct inode *inode, int type);
-int ntfs_set_acl(struct user_namespace *mnt_userns, struct inode *inode,
+int ntfs_set_acl(struct inode *inode,
 		 struct posix_acl *acl, int type);
-int ntfs_init_acl(struct user_namespace *mnt_userns, struct inode *inode,
+int ntfs_init_acl(struct inode *inode,
 		  struct inode *dir);
 #else
 #define ntfs_get_acl NULL
 #define ntfs_set_acl NULL
 #endif
 
-int ntfs_acl_chmod(struct user_namespace *mnt_userns, struct inode *inode);
-int ntfs_permission(struct user_namespace *mnt_userns, struct inode *inode,
+int ntfs_acl_chmod(struct inode *inode);
+int ntfs_permission(struct inode *inode,
 		    int mask);
 ssize_t ntfs_listxattr(struct dentry *dentry, char *buffer, size_t size);
 extern const struct xattr_handler *ntfs_xattr_handlers[];
EOT

patch --ignore-whitespace file.c << 'EOT'
--- a/file.c
+++ b/file.c
@@ -76,7 +76,7 @@ static long ntfs_compat_ioctl(struct file *filp, u32 cmd, unsigned long arg)
 /*
  * inode_operations::getattr
  */
-int ntfs_getattr(struct user_namespace *mnt_userns, const struct path *path,
+int ntfs_getattr(const struct path *path,
 		 struct kstat *stat, u32 request_mask, u32 flags)
 {
 	struct inode *inode = d_inode(path->dentry);
@@ -90,7 +90,7 @@ int ntfs_getattr(struct user_namespace *mnt_userns, const struct path *path,
 
 	stat->attributes_mask |= STATX_ATTR_COMPRESSED | STATX_ATTR_ENCRYPTED;
 
-	generic_fillattr(mnt_userns, inode, stat);
+	generic_fillattr(inode, stat);
 
 	stat->result_mask |= STATX_BTIME;
 	stat->btime = ni->i_crtime;
@@ -614,7 +614,7 @@ out:
 /*
  * inode_operations::setattr
  */
-int ntfs3_setattr(struct user_namespace *mnt_userns, struct dentry *dentry,
+int ntfs3_setattr(struct dentry *dentry,
 		  struct iattr *attr)
 {
 	struct super_block *sb = dentry->d_sb;
@@ -633,7 +633,7 @@ int ntfs3_setattr(struct user_namespace *mnt_userns, struct dentry *dentry,
 		ia_valid = attr->ia_valid;
 	}
 
-	err = setattr_prepare(mnt_userns, dentry, attr);
+	err = setattr_prepare(dentry, attr);
 	if (err)
 		goto out;
 
@@ -658,10 +658,10 @@ int ntfs3_setattr(struct user_namespace *mnt_userns, struct dentry *dentry,
 		ni->ni_flags |= NI_FLAG_UPDATE_PARENT;
 	}
 
-	setattr_copy(mnt_userns, inode, attr);
+	setattr_copy(inode, attr);
 
 	if (mode != inode->i_mode) {
-		err = ntfs_acl_chmod(mnt_userns, inode);
+		err = ntfs_acl_chmod(inode);
 		if (err)
 			goto out;
 
EOT

patch --ignore-whitespace namei.c << 'EOT'
--- a/namei.c
+++ b/namei.c
@@ -102,7 +102,7 @@ static struct dentry *ntfs_lookup(struct inode *dir, struct dentry *dentry,
  *
  * inode_operations::create
  */
-static int ntfs_create(struct user_namespace *mnt_userns, struct inode *dir,
+static int ntfs_create(struct inode *dir,
 		       struct dentry *dentry, umode_t mode, bool excl)
 {
 	struct ntfs_inode *ni = ntfs_i(dir);
@@ -110,7 +110,7 @@ static int ntfs_create(struct user_namespace *mnt_userns, struct inode *dir,
 
 	ni_lock_dir(ni);
 
-	inode = ntfs_create_inode(mnt_userns, dir, dentry, NULL, S_IFREG | mode,
+	inode = ntfs_create_inode(dir, dentry, NULL, S_IFREG | mode,
 				  0, NULL, 0, excl, NULL);
 
 	ni_unlock(ni);
@@ -184,7 +184,7 @@ static int ntfs_unlink(struct inode *dir, struct dentry *dentry)
  *
  * inode_operations::symlink
  */
-static int ntfs_symlink(struct user_namespace *mnt_userns, struct inode *dir,
+static int ntfs_symlink(struct inode *dir,
 			struct dentry *dentry, const char *symname)
 {
 	u32 size = strlen(symname);
@@ -193,7 +193,7 @@ static int ntfs_symlink(struct user_namespace *mnt_userns, struct inode *dir,
 
 	ni_lock_dir(ni);
 
-	inode = ntfs_create_inode(mnt_userns, dir, dentry, NULL, S_IFLNK | 0777,
+	inode = ntfs_create_inode(dir, dentry, NULL, S_IFLNK | 0777,
 				  0, symname, size, 0, NULL);
 
 	ni_unlock(ni);
@@ -206,7 +206,7 @@ static int ntfs_symlink(struct user_namespace *mnt_userns, struct inode *dir,
  *
  * inode_operations::mkdir
  */
-static int ntfs_mkdir(struct user_namespace *mnt_userns, struct inode *dir,
+static int ntfs_mkdir(struct inode *dir,
 		      struct dentry *dentry, umode_t mode)
 {
 	struct inode *inode;
@@ -214,7 +214,7 @@ static int ntfs_mkdir(struct user_namespace *mnt_userns, struct inode *dir,
 
 	ni_lock_dir(ni);
 
-	inode = ntfs_create_inode(mnt_userns, dir, dentry, NULL, S_IFDIR | mode,
+	inode = ntfs_create_inode(dir, dentry, NULL, S_IFDIR | mode,
 				  0, NULL, -1, 0, NULL);
 
 	ni_unlock(ni);
@@ -246,7 +246,7 @@ static int ntfs_rmdir(struct inode *dir, struct dentry *dentry)
  *
  * inode_operations::rename
  */
-static int ntfs_rename(struct user_namespace *mnt_userns, struct inode *old_dir,
+static int ntfs_rename(struct inode *old_dir,
 		       struct dentry *old_dentry, struct inode *new_dir,
 		       struct dentry *new_dentry, u32 flags)
 {
@@ -520,7 +520,7 @@ static int ntfs_atomic_open(struct inode *dir, struct dentry *dentry,
 
 	/*fnd contains tree's path to insert to*/
 	/* TODO: init_user_ns? */
-	inode = ntfs_create_inode(&init_user_ns, dir, dentry, uni, mode, 0,
+	inode = ntfs_create_inode(dir, dentry, uni, mode, 0,
 				  NULL, 0, excl, fnd);
 	err = IS_ERR(inode) ? PTR_ERR(inode)
 			    : finish_open(file, dentry, ntfs_file_open);
EOT

patch --ignore-whitespace xattr.c << 'EOT'
--- a/xattr.c
+++ b/xattr.c
@@ -473,7 +473,7 @@ static inline void ntfs_posix_acl_release(struct posix_acl *acl)
 		kfree(acl);
 }
 
-static struct posix_acl *ntfs_get_acl_ex(struct user_namespace *mnt_userns,
+static struct posix_acl *ntfs_get_acl_ex(
 					 struct inode *inode, int type,
 					 int locked)
 {
@@ -509,7 +509,7 @@ static struct posix_acl *ntfs_get_acl_ex(struct user_namespace *mnt_userns,
 
 	/* Translate extended attribute to acl */
 	if (err > 0) {
-		acl = posix_acl_from_xattr(mnt_userns, buf, err);
+		acl = posix_acl_from_xattr(&init_user_ns, buf, err);
 		if (!IS_ERR(acl))
 			set_cached_acl(inode, type, acl);
 	} else {
@@ -529,10 +529,10 @@ static struct posix_acl *ntfs_get_acl_ex(struct user_namespace *mnt_userns,
 struct posix_acl *ntfs_get_acl(struct inode *inode, int type)
 {
 	/* TODO: init_user_ns? */
-	return ntfs_get_acl_ex(&init_user_ns, inode, type, 0);
+	return ntfs_get_acl_ex(inode, type, 0);
 }
 
-static noinline int ntfs_set_acl_ex(struct user_namespace *mnt_userns,
+static noinline int ntfs_set_acl_ex(
 				    struct inode *inode, struct posix_acl *acl,
 				    int type, int locked)
 {
@@ -590,7 +590,7 @@ static noinline int ntfs_set_acl_ex(struct user_namespace *mnt_userns,
 	if (!value)
 		return -ENOMEM;
 
-	err = posix_acl_to_xattr(mnt_userns, acl, value, size);
+	err = posix_acl_to_xattr(&init_user_ns, acl, value, size);
 	if (err)
 		goto out;
 
@@ -614,13 +614,13 @@ out:
  *
  * inode_operations::set_acl
  */
-int ntfs_set_acl(struct user_namespace *mnt_userns, struct inode *inode,
+int ntfs_set_acl(struct inode *inode,
 		 struct posix_acl *acl, int type)
 {
-	return ntfs_set_acl_ex(mnt_userns, inode, acl, type, 0);
+	return ntfs_set_acl_ex(inode, acl, type, 0);
 }
 
-static int ntfs_xattr_get_acl(struct user_namespace *mnt_userns,
+static int ntfs_xattr_get_acl(
 			      struct inode *inode, int type, void *buffer,
 			      size_t size)
 {
@@ -637,13 +637,13 @@ static int ntfs_xattr_get_acl(struct user_namespace *mnt_userns,
 	if (!acl)
 		return -ENODATA;
 
-	err = posix_acl_to_xattr(mnt_userns, acl, buffer, size);
+	err = posix_acl_to_xattr(&init_user_ns, acl, buffer, size);
 	ntfs_posix_acl_release(acl);
 
 	return err;
 }
 
-static int ntfs_xattr_set_acl(struct user_namespace *mnt_userns,
+static int ntfs_xattr_set_acl(
 			      struct inode *inode, int type, const void *value,
 			      size_t size)
 {
@@ -653,23 +653,23 @@ static int ntfs_xattr_set_acl(struct user_namespace *mnt_userns,
 	if (!(inode->i_sb->s_flags & SB_POSIXACL))
 		return -EOPNOTSUPP;
 
-	if (!inode_owner_or_capable(mnt_userns, inode))
+	if (!inode_owner_or_capable(inode))
 		return -EPERM;
 
 	if (!value)
 		return 0;
 
-	acl = posix_acl_from_xattr(mnt_userns, value, size);
+	acl = posix_acl_from_xattr(&init_user_ns, value, size);
 	if (IS_ERR(acl))
 		return PTR_ERR(acl);
 
 	if (acl) {
-		err = posix_acl_valid(mnt_userns, acl);
+		err = posix_acl_valid(&init_user_ns, acl);
 		if (err)
 			goto release_and_out;
 	}
 
-	err = ntfs_set_acl(mnt_userns, inode, acl, type);
+	err = ntfs_set_acl(inode, acl, type);
 
 release_and_out:
 	ntfs_posix_acl_release(acl);
@@ -679,7 +679,7 @@ release_and_out:
 /*
  * Initialize the ACLs of a new inode. Called from ntfs_create_inode.
  */
-int ntfs_init_acl(struct user_namespace *mnt_userns, struct inode *inode,
+int ntfs_init_acl(struct inode *inode,
 		  struct inode *dir)
 {
 	struct posix_acl *default_acl, *acl;
@@ -691,7 +691,7 @@ int ntfs_init_acl(struct user_namespace *mnt_userns, struct inode *inode,
 	 */
 	inode->i_default_acl = NULL;
 
-	default_acl = ntfs_get_acl_ex(mnt_userns, dir, ACL_TYPE_DEFAULT, 1);
+	default_acl = ntfs_get_acl_ex(dir, ACL_TYPE_DEFAULT, 1);
 
 	if (!default_acl || default_acl == ERR_PTR(-EOPNOTSUPP)) {
 		inode->i_mode &= ~current_umask();
@@ -719,13 +719,13 @@ int ntfs_init_acl(struct user_namespace *mnt_userns, struct inode *inode,
 	}
 
 	if (default_acl)
-		err = ntfs_set_acl_ex(mnt_userns, inode, default_acl,
+		err = ntfs_set_acl_ex(inode, default_acl,
 				      ACL_TYPE_DEFAULT, 1);
 
 	if (!acl)
 		inode->i_acl = NULL;
 	else if (!err)
-		err = ntfs_set_acl_ex(mnt_userns, inode, acl, ACL_TYPE_ACCESS,
+		err = ntfs_set_acl_ex(inode, acl, ACL_TYPE_ACCESS,
 				      1);
 
 	posix_acl_release(acl);
@@ -742,7 +742,7 @@ out:
  *
  * helper for 'ntfs3_setattr'
  */
-int ntfs_acl_chmod(struct user_namespace *mnt_userns, struct inode *inode)
+int ntfs_acl_chmod(struct inode *inode)
 {
 	struct super_block *sb = inode->i_sb;
 
@@ -752,7 +752,7 @@ int ntfs_acl_chmod(struct user_namespace *mnt_userns, struct inode *inode)
 	if (S_ISLNK(inode->i_mode))
 		return -EOPNOTSUPP;
 
-	return posix_acl_chmod(mnt_userns, inode, inode->i_mode);
+	return posix_acl_chmod(inode, inode->i_mode);
 }
 
 /*
@@ -760,7 +760,7 @@ int ntfs_acl_chmod(struct user_namespace *mnt_userns, struct inode *inode)
  *
  * inode_operations::permission
  */
-int ntfs_permission(struct user_namespace *mnt_userns, struct inode *inode,
+int ntfs_permission(struct inode *inode,
 		    int mask)
 {
 	if (ntfs_sb(inode->i_sb)->options.no_acs_rules) {
@@ -768,7 +768,7 @@ int ntfs_permission(struct user_namespace *mnt_userns, struct inode *inode,
 		return 0;
 	}
 
-	return generic_permission(mnt_userns, inode, mask);
+	return generic_permission(inode, mask);
 }
 
 /*
@@ -882,7 +882,7 @@ static int ntfs_getxattr(const struct xattr_handler *handler, struct dentry *de,
 		     sizeof(XATTR_NAME_POSIX_ACL_DEFAULT)))) {
 		/* TODO: init_user_ns? */
 		err = ntfs_xattr_get_acl(
-			&init_user_ns, inode,
+			inode,
 			name_len == sizeof(XATTR_NAME_POSIX_ACL_ACCESS) - 1
 				? ACL_TYPE_ACCESS
 				: ACL_TYPE_DEFAULT,
@@ -903,7 +903,6 @@ out:
  * inode_operations::setxattr
  */
 static noinline int ntfs_setxattr(const struct xattr_handler *handler,
-				  struct user_namespace *mnt_userns,
 				  struct dentry *de, struct inode *inode,
 				  const char *name, const void *value,
 				  size_t size, int flags)
@@ -1013,7 +1012,7 @@ set_new_fa:
 		     sizeof(XATTR_NAME_POSIX_ACL_DEFAULT)))) {
 		/* TODO: init_user_ns? */
 		err = ntfs_xattr_set_acl(
-			&init_user_ns, inode,
+			inode,
 			name_len == sizeof(XATTR_NAME_POSIX_ACL_ACCESS) - 1
 				? ACL_TYPE_ACCESS
 				: ACL_TYPE_DEFAULT,
EOT

patch --ignore-whitespace inode.c << 'EOT'
--- a/inode.c
+++ b/inode.c
@@ -696,36 +696,6 @@ static int ntfs_readpage(struct file *file, struct page *page)
 	return mpage_readpage(page, ntfs_get_block);
 }
 
-static void ntfs_readahead(struct readahead_control *rac)
-{
-	struct address_space *mapping = rac->mapping;
-	struct inode *inode = mapping->host;
-	struct ntfs_inode *ni = ntfs_i(inode);
-	u64 valid;
-	loff_t pos;
-
-	if (is_resident(ni)) {
-		/* no readahead for resident */
-		return;
-	}
-
-	if (is_compressed(ni)) {
-		/* no readahead for compressed */
-		return;
-	}
-
-	valid = ni->i_valid;
-	pos = readahead_pos(rac);
-
-	if (valid < i_size_read(inode) && pos <= valid &&
-	    valid < pos + readahead_length(rac)) {
-		/* range cross 'valid'. read it page by page */
-		return;
-	}
-
-	mpage_readahead(rac, ntfs_get_block);
-}
-
 static int ntfs_get_block_direct_IO_R(struct inode *inode, sector_t iblock,
 				      struct buffer_head *bh_result, int create)
 {
@@ -1176,7 +1146,7 @@ out:
 	return ERR_PTR(err);
 }
 
-struct inode *ntfs_create_inode(struct user_namespace *mnt_userns,
+struct inode *ntfs_create_inode(
 				struct inode *dir, struct dentry *dentry,
 				const struct cpu_str *uni, umode_t mode,
 				dev_t dev, const char *symname, u32 size,
@@ -1577,7 +1547,7 @@ struct inode *ntfs_create_inode(struct user_namespace *mnt_userns,
 
 #ifdef CONFIG_NTFS3_FS_POSIX_ACL
 	if (!is_link && (sb->s_flags & SB_POSIXACL)) {
-		err = ntfs_init_acl(mnt_userns, inode, dir);
+		err = ntfs_init_acl(inode, dir);
 		if (err)
 			goto out6;
 	} else
@@ -2018,7 +1988,6 @@ const struct inode_operations ntfs_link_inode_operations = {
 
 const struct address_space_operations ntfs_aops = {
 	.readpage = ntfs_readpage,
-	.readahead = ntfs_readahead,
 	.writepage = ntfs_writepage,
 	.writepages = ntfs_writepages,
 	.write_begin = ntfs_write_begin,
@@ -2029,5 +1998,4 @@ const struct address_space_operations ntfs_aops = {
 
 const struct address_space_operations ntfs_aops_cmpr = {
 	.readpage = ntfs_readpage,
-	.readahead = ntfs_readahead,
 };
EOT

wget -O dkms.conf https://aur.archlinux.org/cgit/aur.git/plain/dkms.conf?h=ntfs3-dkms
echo 'MODULE_INFO(intree, "Y");' >> super.c
dkms add -m ${pkgname} -v ${pkgver}
dkms build -m ${pkgname} -v ${pkgver}
dkms install -m ${pkgname} -v ${pkgver}
echo Installation succeeded.
{% endhighlight %}

The script above downloads the files in the appropiate directory, and then uses the DKMS to build and install the module. Also, it should execute very fast, at most 1 minute with a decent Internet connection, this including the actual compilation which does not take long at all.

Beware that I add a line to the module that marks it as part of the kernel tree, so that when the module is loaded, the kernel does not get tainted. I do this because a tainted kernel loses some debugging functionality. While it will be in tree at some point, it is not at the moment. If you experience problems with your kernel, consider it tainted and do not submit reports with this module loaded. Instead, make sure the module does not get loaded, reboot the system, catch the problem and submit a report with an untainted kernel. You can verify the condition of your kernel by issuing:

```
~ cat /proc/sys/kernel/tainted
```

Decode the value as explained [here](https://www.kernel.org/doc/html/latest/admin-guide/tainted-kernels.html). If you don't want this behavior, take out this line from the script above: `echo 'MODULE_INFO(intree, "Y");' >> super.c`.

In the end, what you have to do, in order to mount an NTFS partition using this driver, is to explicitly specify the type when issuing the mount command, like so:

```
# mount -t ntfs3 /dev/sdb1 /mnt
```

If everything went ok, the drive should be mounted (check in `lsblk` and `findmnt`). Should any error occur, start by first taking a look in `dmesg`.

Based on my use case (sharing via Samba), plus the documentation of this driver (available at `/usr/src/ntfs3-17.0.0/ntfs3.rst`) and of the commercial variant (available [here](https://dl.paragon-software.com/doc/NTFS_HFS_linux_user_manual.pdf); the commercial variant is different from this driver but has some similarities), I mount my drives using this command line that I deemed optimal for my situation:

```
# mount -t ntfs3 -o umask=000,fmask=000,dmask=000,discard,force,prealloc,no_acs_rules,acl /dev/sdb1 /mnt
```

To avoid having to specify the type, the comments on the AUR package suggest using an udev rule. I personally haven't used that, as I mount my disks at boot using `/etc/fstab` anyway, but it definitely should work and is worth a try. They also offer suggestions for enabling easy mounting in desktop environments (I use Raspbian Lite, so headless).

Now, do some testing. For me, I easily saturate the link (around 115MB/s), which is awesome. On par with ext4, at least for the kind of speeds a gigabit connection supports, which is what I was looking for. Indeed, a proper solution. And this without any overclock on the Pi, which should not be required (my Pi4 is passively cooled, so I try not to push it excessively - I value quietness more).

I also provide an uninstall script, in case you want to completely get rid of this from your system (again, run as root and unmount all the disks using the driver so it can be unloaded from the kernel):

{% highlight bash%}
#!/bin/bash
pkgname=ntfs3
pkgver=26.0.0
prefix=/usr/src

rmmod ntfs3
dkms uninstall -m ${pkgname} -v ${pkgver}
dkms remove ${pkgname}/${pkgver} --all
rm -rf ${prefix}/${pkgname}-${pkgver}
echo Uninstall succeeded.
{% endhighlight %}

That's it, hope it serves you well. Hopefully, this will be integrated as soon as possible in the kernel and eventually reach the Pi as well, as this is pretty big. Finally a proper file system that works best with both Windows and Linux and with no headaches. I mean, I don't want to be the one to suggest it, but I am sure someone will soon boot Linux off NTFS. Maybe have a single drive with both Windows and Linux, with home and My Documents mapped to the same folder and boot in either OS using two different entries in the EFI boot menu. Interesting times ahead. This module definitely is an incredible contribution, I can't wait for it to ship - probably with 5.12, if not 5.11.

P.S. You can use `/etc/fstab` to mount the filesystem instead of running the mount command every time, especially if you want to have the drive mounted automatically at boot. If you do not know what to write there, which I don't know as well, again, Arch has a nifty utility to help us, courtesy of their excellent installation guide: `genfstab` - this generates an fstab file from the current mounts in the system. To install it on the Pi, do:

```
# apt install arch-install-scripts
```

Indeed, there is a Debian package for that. Then, mount the partition using ntfs3 as shown above, after which generate an fstab:

```
# genfstab -U /
```

This will write the fstab at stdout. Just copy the line containing your partition that is mounted using the ntfs3 driver and put it into your real fstab. It's that easy.

P.P.S. If you simply want to build the module manually in some folder where you get the files, execute this:

```
~ KVERSION=$(uname -r) CONFIG_NTFS3_FS=m CONFIG_NTFS3_LZX_XPRESS=y CONFIG_NTFS3_FS_POSIX_ACL=y make KDIR=/lib/modules/$(uname -r)/build
```

To load the module in the kernel, use `insmod` in the same folder:

```
# insmod ntfs3.ko
```

To remove it from the kernel, use:

```
# rmmod ntfs3
```

P.P.P.S. Of course, the script is universal, it works on any distro and you can simply delete the part that patches out the readahead support if your kernel is 5.8+.