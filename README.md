# UEFI環境用 Nutanix CE AVH インストールイメージ作成方法

# 目的
Nutanix CE AVHはUEFIからbootが出来ない。そのため2TBを超えるHDDが使用出来ないなど実用性に問題がある。
そこで、Nutanix CEの起動ベースとなるCentOS 7のオリジナルISOからUEFIブートデータを作成する。

# 参考URL
- https://yukisaki.hatenablog.com/entry/2019/12/23/104546
- https://github.com/abbbi/nutanix_uefi
- https://access.redhat.com/documentation/ja-jp/red_hat_enterprise_linux/7/html/system_administrators_guide/ch-working_with_the_grub_2_boot_loader

# 用意したもの
- ce-2019.11.22-stable.img
- CentOS-7-x86_64-DVD-2003.iso

# 作業環境
CentOS-7-x86_64-DVD-2003.iso を最小構成でインストールした環境において、rootユーザーで作業する。

最初にvfatフォーマットをするために`dosfstools`をインスートール ※参照元のページはタイプミスがあるので注意
``` bash
yum install dosfstools  
```

# Nutanix CE イメージファイルにUEFIブートのための領域を追加
ダウンロードしたimgファイルの容量を確認する。
``` bash
# cfdisk ce-2019.11.22-stable.img 
```
```
                           cfdisk (util-linux 2.23.2)

               ディスクドライブ: ce-2019.11.22-stable.img
                    サイズ: 7444889600 バイト, 7444 MB
  ヘッド: 43   トラック当たりのセクタ: 32   シリンダ: 10567
  <省略>  
```
サイズが`7444889600` バイトであることが確認できた。

# オリジナルのイメージファイルへ拡張スペースを作成
調べたサイズを用い、その直後に40Mb分の拡張スペースを作成する。
``` bash
# fallocate -o 7444889600 -l 40m ce-2019.11.22-stable.img
```

# 拡張スペースにパーティションを作成する
``` bash
# cfdisk ./ce-2019.11.22-stable.img
```

```
                           cfdisk (util-linux 2.23.2)

              ディスクドライブ: ./ce-2019.11.22-stable.img
                    サイズ: 7486832640 バイト, 7486 MB
  ヘッド: 43   トラック当たりのセクタ: 32   シリンダ: 10626

    名前        フラグ     パーティショFSタイプ        [ラベル]       サイズ (MB
)------------------------------------------------------------------------------
                            基本/論理 空き領域                             1.05*
    ce-2019.11.2ブートle.img基本領域  ext4                              7443.85*
                            基本/論理 空き領域                            41.95*


     [ヘルプ ]     [新規作成]    [  表示  ]    [  終了  ]    [  単位  ]
     [書き込み]

    空きパーティションから新しくパーティションを作成

```
十字キーで下記の通り選んで、Enterで実行
- [新規作成] 
- [基本領域]
- サイズ (MB 単位): 41.94※ディフォルト
- [ブート可]
- [書き込み]

# イメージファイルにアクセスする準備
``` bash
# kpartx -av ./ce-2019.11.22-stable.img
```
```
add map loop0p1 (253:3): 0 14538752 linear /dev/loop0 2048
add map loop0p2 (253:4): 0 81920 linear /dev/loop0 14540800
```

# 拡張したパーティションをvfatでフォーマット
``` bash
# kpartx -av ./ce-2019.11.22-stable.img
```
以下の通り表示される
```
mkfs.fat 3.0.20 (12 Jun 2013)
unable to get drive geometry, using default 255/63
```
確認は
``` bash
# cfdisk ./ce-2019.11.22-stable.img
```
```
                           cfdisk (util-linux 2.23.2)

               ディスクドライブ: ce-2019.11.22-stable.img
                    サイズ: 7486832640 バイト, 7486 MB
  ヘッド: 43   トラック当たりのセクタ: 32   シリンダ: 10626

    名前        フラグ     パーティショFSタイプ        [ラベル]       サイズ (MB
)------------------------------------------------------------------------------
                            基本/論理 空き領域                             1.05*
    ce-2019.11.2ブートle.img基本領域  ext4				7443.85*
    ce-2019.11.2ブートle.img基本領域  vfat                                41.95*

```
２個目がvfatになっていればOK

# イメージをマウント
ここまで準備したNutanixCEのイメージと、CentOSのISOをマウントする。
``` bash
# mkdir /tmp/nutanix_vfat
# mkdir /tmp/nutanix_root
# mkdir /tmp/centos
# mount /dev/mapper/loop0p1 /tmp/nutanix_root
# mount /dev/mapper/loop0p2 /tmp/nutanix_vfat
# mount -o loop CentOS-7-x86_64-DVD-2003.iso /tmp/centos
```

# CentOSのメディアから、NutanixCEのイメージにEFIデータをコピー
``` bash
# cp -r /tmp/centos/EFI /tmp/nutanix_vfat/
```

# grub.cfgを変更する
変更先
```
/tmp/nutanix_vfat/EFI/BOOT/grub.cfg
```

参考元
```
/tmp/nutanix_root/boot/grub2/grub.cfg
```
CentOS 7の範囲を削除する。
参考元から下記の範囲を変更先へまずコピーする。

``` 
menuentry 'Nutanix Community Edition AHV (4.4.77-1.el7.nutanix.20191030.415.x86_64) 7 (Core)' --class nutanix --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-4.4.77-1.el7.nutanix.20191030.415.x86_64-advanced-6e28729c-5954-44c8-87a9-70efa532e7d2' {
	load_video
	set gfxpayload=keep
	insmod gzio
	insmod part_msdos
	insmod ext2
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root  6e28729c-5954-44c8-87a9-70efa532e7d2
	else
	  search --no-floppy --fs-uuid --set=root 6e28729c-5954-44c8-87a9-70efa532e7d2
	fi
	linux16 /boot/vmlinuz-4.4.77-1.el7.nutanix.20191030.415.x86_64 root=UUID=6e28729c-5954-44c8-87a9-70efa532e7d2 ro crashkernel=128M rhgb quiet hugepages=0 intel_iommu=on,igfx_off iommu=pt elevator=noop vga=791 vfio_iommu_type1.allow_unsafe_interrupts=1 
	initrd16 /boot/initramfs-4.4.77-1.el7.nutanix.20191030.415.x86_64.img
}
```

下記の２箇所を置換する。
```
linux16 -> linuxefi
initrd16 -> initrdefi
```

下記も書き換えると起動がスムーズ
```
set default="0" -> set default="1"
```

全体内容
``` bash
# cat /tmp/nutanix_vfat/EFI/BOOT/grub.cfg
```
```
set default="0"

function load_video {
  insmod efi_gop
  insmod efi_uga
  insmod video_bochs
  insmod video_cirrus
  insmod all_video
}
menuentry 'Nutanix Community Edition AHV (4.4.77-1.el7.nutanix.20191030.415.x86_64) 7 (Core)' --class nutanix --class gnu-linux --class gnu --class os --unrestricted $menuentry_id_option 'gnulinux-4.4.77-1.el7.nutanix.20191030.415.x86_64-advanced-6e28729c-5954-44c8-87a9-70efa532e7d2' {
	load_video
	set gfxpayload=keep
	insmod gzio
	insmod part_msdos
	insmod ext2
	if [ x$feature_platform_search_hint = xy ]; then
	  search --no-floppy --fs-uuid --set=root  6e28729c-5954-44c8-87a9-70efa532e7d2
	else
	  search --no-floppy --fs-uuid --set=root 6e28729c-5954-44c8-87a9-70efa532e7d2
	fi
	linuxefi /boot/vmlinuz-4.4.77-1.el7.nutanix.20191030.415.x86_64 root=UUID=6e28729c-5954-44c8-87a9-70efa532e7d2 ro crashkernel=128M rhgb quiet hugepages=0 intel_iommu=on,igfx_off iommu=pt elevator=noop vga=791 vfio_iommu_type1.allow_unsafe_interrupts=1 
	initrdefi /boot/initramfs-4.4.77-1.el7.nutanix.20191030.415.x86_64.img
}
```

書き換え完了！

