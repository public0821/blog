# image(镜像)是什么

上一篇介绍了hello-world的大概流程，那么hello-world的image里面到底包含了些什么呢？里面的格式是怎么样的呢？

## image包含的内容

image的标准现在由opencontainer负责维护，地址为[image-spec](https://github.com/opencontainers/image-spec/blob/master/spec.md)。一个image由[manifest](https://github.com/opencontainers/image-spec/blob/master/manifest.md), [image index](https://github.com/opencontainers/image-spec/blob/master/image-index.md) (可选), [filesystem layers](https://github.com/opencontainers/image-spec/blob/master/layer.md)和[configuration](https://github.com/opencontainers/image-spec/blob/master/config.md)四部分组成。

### 关系图
先来看看构成image的四部分的关系图：
```
                    +-----------------------+
                    | Image Index(optional) |
                    +-----------------------+
                               |
                               | 1..*
                               ↓
                    +----------------------+
                    |    Image Manifest    |
                    +----------------------+
                               |
                     1..1      |     1..*
               +---------------+--------------+
               |                              |
               ↓                              ↓
       +--------------+             +-------------------+
       | Image Config |             | Filesystem Layers |
       +--------------+             +-------------------+
```

* Image Index和Manifest的关系是"1..*"，表示它们是一对多的关系
* Image Manifest和Config的关系是"1..1"，表示它们是一对一的关系
* Image Manifest和Filesystem Layers是一对多的关系

下面分别介绍它们各自都包含了哪些内容。

### Filesystem Layers
标准中的Filesystem Layer包含两部分信息，分别是包含的内容和打包格式

#### 包含的内容
每个filesystem layer都包含了在上一个layer上的改动情况，主要包含三方面的内容：

* 变化类型：是增加、修改还是删除了文件
* 文件类型：每个变化发生在哪种文件类型上
* 文件属性：文件的修改时间、用户ID、组ID、RWX权限等

比如在某一层增加了一个文件，那么这一层所包含的内容就是增加的这个文件的数据以及它的属性，具体的细节请参考[标准文档](https://github.com/opencontainers/image-spec/blob/master/layer.md)，下一篇中会给出一个简单示例。

#### 打包格式
最终每个layer都会打包成一个文件，这个文件的格式可以是tar和tar+gzip两种中的一种。

对于这两种不同类型的文件格式，标准定义了两个新的media types，分别是application/vnd.oci.image.layer.v1.tar和application/vnd.oci.image.layer.v1.tar+gzip。

同时标准还定义了application/vnd.oci.image.layer.nondistributable.v1.tar和application/vnd.oci.image.layer.nondistributable.v1.tar+gzip这两种对应于nondistributable的格式，其实这两种格式和前两种格式包含的内容是一样的，只是用不同的类型名称来区分它们的用途，对于名称中有nondistributable的layer，标准要求这种类型的layer不能上传，只能下载。

>做过web开发的程序员对media type应该比较熟悉，简单点说，就是当客户端用http协议下载一个文件的时候，需要在http的首部带上Accept字段，告诉服务器端它支持哪些类型的文件，服务器返回文件的时候，需要在http的首部带上Content-Type字段，告诉客户端返回文件的类型，如```Accept: text/html,application/xml```和```Content-Type: text/html```。

### Image Config
image config就是一个json文件，它的media type是```application/vnd.oci.image.config.v1+json```，这个json文件包含了对这个image的描述。先看看官方网站给的例子：

```
{
    "created": "2015-10-31T22:22:56.015925234Z",
    "author": "Alyssa P. Hacker <alyspdev@example.com>",
    "architecture": "amd64",
    "os": "linux",
    "config": {
        "User": "alice",
        "ExposedPorts": {
            "8080/tcp": {}
        },
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "FOO=oci_is_a",
            "BAR=well_written_spec"
        ],
        "Entrypoint": [
            "/bin/my-app-binary"
        ],
        "Cmd": [
            "--foreground",
            "--config",
            "/etc/my-app.d/default.cfg"
        ],
        "Volumes": {
            "/var/job-result-data": {},
            "/var/log/my-app-logs": {}
        },
        "WorkingDir": "/home/alice",
        "Labels": {
            "com.example.project.git.url": "https://example.com/project.git",
            "com.example.project.git.commit": "45a939b2999782a3f005621a8d0f29aa387e1d6b"
        }
    },
    "rootfs": {
      "diff_ids": [
        "sha256:c6f988f4874bb0add23a778f753c65efe992244e148a1d2ec2a8b664fb66bbd1",
        "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef"
      ],
      "type": "layers"
    },
    "history": [
      {
        "created": "2015-10-31T22:22:54.690851953Z",
        "created_by": "/bin/sh -c #(nop) ADD file:a3bc1e842b69636f9df5256c49c5374fb4eef1e281fe3f282c65fb853ee171c5 in /"
      },
      {
        "created": "2015-10-31T22:22:55.613815829Z",
        "created_by": "/bin/sh -c #(nop) CMD [\"sh\"]",
        "empty_layer": true
      }
    ]
}
```

这里只介绍几个比较重要的属性，其它的请参考[标准文档](https://github.com/opencontainers/image-spec/blob/master/config.md)

* **architecture**：CPU架构类型，现在大部分都是amd64，不过arm64估计会慢慢多起来
* **os**：操作系统，主要是linux
* **config**：当根据这个image启动container时，config里面的配置就是运行container时的默认参数，在后续介绍runtime的时候再仔细介绍每一项的意义
* **rootfs**：指定了image所包含的filesystem layers，type的值必须是layers，diff_ids包含了layer的列表，从上到下分别对应从底到上的layer，每一个sha256就是每层layer对应tar包的sha256码

### manifest
manifest也是一个json文件，media type为```application/vnd.oci.image.manifest.v1+json```，这个文件包含了对前面filesystem layers和image config的描述，一看官方网站给出的示例就明白了：
```
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 7023,
    "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7"
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 32654,
      "digest": "sha256:e692418e4cbaf90ca69d05a66403747baa33ee08806650b51fab815ad7fc331f"
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 16724,
      "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b"
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 73109,
      "digest": "sha256:ec4b8955958665577945c89419d1af06b5f7636b4ac3da7f12184802ad867736"
    }
  ],
  "annotations": {
    "com.example.key1": "value1",
    "com.example.key2": "value2"
  }
}
```

* config里面包含了对image config文件的描述，有media type，文件大小，以及sha256码
* layers包含了对每一个layer的描述，和对config文件的描述一样，也包含了media type，文件大小，以及sha256码

>这里layer的sha256和image config文件中的diff_ids有可能不一样，比如这里的layer文件格式是tar+gzip，那么这里的sha256就是tar+gzip包的sha256码，而diff_ids是tar+gzip解压后tar文件的sha256码

### Image Index(可选)
image index也是个json文件，media type是```application/vnd.oci.image.index.v1+json```。

其实到manifest为止，已经有了整个image的完整描述，为什么还需要image index这个文件呢？主要原因是manifest描述的image只能支持一个平台，没法支持多个平台，加上index文件的目的就是让这个image支持多个平台，看官方给的示例：

```
{
  "schemaVersion": 2,
  "manifests": [
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 7143,
      "digest": "sha256:e692418e4cbaf90ca69d05a66403747baa33ee08806650b51fab815ad7fc331f",
      "platform": {
        "architecture": "ppc64le",
        "os": "linux"
      }
    },
    {
      "mediaType": "application/vnd.oci.image.manifest.v1+json",
      "size": 7682,
      "digest": "sha256:5b0bcabd1ed22e9fb1310cf6c2dec7cdef19f0ad69efa1f392e94a4333501270",
      "platform": {
        "architecture": "amd64",
        "os": "linux",
        "os.features": [
          "sse4"
        ]
      }
    }
  ],
  "annotations": {
    "com.example.key1": "value1",
    "com.example.key2": "value2"
  }
}
```
index文件包含了对manifest的描述，包括每个manifest的media type，文件大小，sha256码，支持的平台以及平台特殊的配置。

比如ubuntu想让它的image支持amd64和arm64平台，于是它在两个平台上都进行了编译，然后将两个平台的layer都放到这个filesystem layers里面，然后写两个config文件和两个manifest文件，再加上这样一个描述不同平台manifest的index文件，就可以让这个image支持两个平台了，两个平台的用户可以使用同样的命令得到自己平台想要的那些layer。

>据我所知，目前docker还不支持该文件，估计将不同平台的image分开管理要比放在一起要好一点

## image layout
上面介绍了image所包含的内容，在开始介绍layout之前，先来回顾一下上一篇介绍hello-world时提到的从register服务器拉image的过程：

* 首先获取image的manifests
* 根据manifests文件中config的sha256码，得到image config文件
* 遍历manifests里面的所有layer，根据其sha256码在本地找，如果找到对应的layer，则跳过，否则从服务器取相应layer的压缩包
* 等上面的所有步骤完成后，就会拼出完整的image

从上面的过程中可以看出，我们从服务器上取image的时候不需要知道image manifests和config文件的名字，也不需要知道layer压缩包的名字。

那么image从服务器拉下来后，在本地应该怎么存储呢？文件名称和目录结构应该是怎样的呢？OCI也有相应的标准，名字叫[image layout](https://github.com/opencontainers/image-spec/blob/master/image-layout.md)，有了这样的标准之后，我们就可以将整个image打成一个包，方便的在不同机器，不同容器平台之间导入导出。


>docker对OCI image layout的支持还在开发中，相关动态请关注：[Support OCI image layout in docker save/load](https://github.com/moby/moby/pull/26369)

## 下载image
为了快速的构建container的rootfs，docker在本地有它自己的一套image管理方式，有自己的layout，并且目前```docker save```命令也不支持导出OCI格式的image，只能导出docker自己的格式，所以我们只能借助其它的工具得到OCI格式的image。

这里我们用[skopeo](https://github.com/projectatomic/skopeo)来演示一下从docker hub上拉取image，并把它在本地存成OCI的layout。

```
#这里的所有命令在debian 8.6的环节上运行通过
$ git clone https://github.com/projectatomic/skopeo $GOPATH/src/github.com/projectatomic/skopeo
$ sudo apt-get install libgpgme11-dev libdevmapper-dev btrfs-tools go-md2man
$ cd $GOPATH/src/github.com/projectatomic/skopeo 
$ make binary-local
$ sudo make install
$ skopeo copy docker://busybox oci:busybox-oci
$ tree busybox-oci/
busybox-oci/
├── blobs
│   └── sha256
│       ├── 636fcf0bc8246e08d2df4771dc764d35ea50428b8dfaa904773b0707cb4f6303
│       ├── 7520415ce76232cdd62ecc345cea5ea44f5b6b144dc62351f2cd2b08382532a3
│       └── 9b8e3ce88f3a2aaa478cfe613632f38d27be5eddaa002a719fa1bfa9ff4f7f63
├── oci-layout
└── refs
    └── latest
```

## 参考

* [Image Format Specification](https://github.com/opencontainers/image-spec/blob/master/spec.md)
