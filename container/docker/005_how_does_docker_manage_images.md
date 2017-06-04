    defaultPidFile  = "/var/run/docker.pid"
    defaultGraph    = "/var/lib/docker"
    defaultExecRoot = "/var/run/docker"

--graph, -g /var/lib/docker Root of the Docker runtime
--exec-root /var/run/docker Root directory for execution state files


root@dev:/var/lib/docker# ls
aufs  containers  image  network  plugins  swarm  tmp  trust  volumes

tmp: temperary directory for pull/import/build image

imageRoot := filepath.Join(config.Root, "image", graphDriver)  ///var/lib/docker/image/aufs

MetadataStorePathTemplate: filepath.Join(config.Root, "image", "%s", "layerdb"),  ///var/lib/docker/image/aufs/layerdb

ifs, err := image.NewFSStoreBackend(filepath.Join(imageRoot, "imagedb")) ///var/lib/docker/image/aufs/imagedb



d.imageStore, err = image.NewImageStore(ifs, d.layerStore)


trustDir := filepath.Join(config.Root, "trust") ///var/lib/docker/trust


distributionMetadataStore, err := dmetadata.NewFSMetadataStore(filepath.Join(imageRoot, "distribution")) ///var/lib/docker/image/aufs/distribution

referenceStore, err := reference.NewReferenceStore(filepath.Join(imageRoot, "repositories.json"))  ///var/lib/docker/image/aufs/repositories.json



imagePullConfig := &distribution.ImagePullConfig{
        Config: distribution.Config{
            MetaHeaders:      metaHeaders,
            AuthConfig:       authConfig,
            ProgressOutput:   progress.ChanOutput(progressChan),
            RegistryService:  daemon.RegistryService,
            ImageEventLogger: daemon.LogImageEvent,
            MetadataStore:    daemon.distributionMetadataStore,
            ImageStore:       distribution.NewImageConfigStoreFromStore(daemon.imageStore),
            ReferenceStore:   daemon.referenceStore,
        },
        DownloadManager: daemon.downloadManager,
        Schema2Types:    distribution.ImageTypes,
    }



### 先拉取manifest，得到config和layers信息
manifest的定义以及包含的内容可以参考[走进docker(02)：image(镜像)是什么？]()里面关于OCI manifest的介绍，docker用的manifest可能有一些小小的差别，但基本的内容和格式都是一样的。

### 根据config的sha256在本地找有没有对应的image，config的sha256就是image的id

### 根据manifest中config的 type + mediasha256去服务器拉取config.json

### 根据manifest中的layer信息（media type + sha256）去download每一个layer

### 一般layer都是压缩过的，拿到后进行解压，如果解压后的sha256码和config.json文件中的rootfs里面的diff_ids相等，说明没问题

### 将config.json写入目录imagedb/content/sha256/，文件名称就是其sha256，即image id

### manifest中layer的degist和config.json文件中diffid的对应关系存在distribution/diffid-by-digest/sha256/

### diffid和digest的对应关系存在distribution/v2metadata-by-diffid/：
[
    {
        "Digest": "sha256:0062f774e9942f61d13928855ab8111adc27def6f41bd6f7902c329ec836882b",
        "HMAC": "",
        "SourceRepository": "docker.io/library/ubuntu"
    }
]

### 每一层都有一个本地生成的随机cacheid，对于不同的backend driver来说，有不同的文件组织形式diff  layers  mnt

layers下面存放的是layer的父子关系，文件名称都是cacheid，文件里面的内容也是cacheid，包含自己layer的所有parent layer，文件最后面的是最上面的parent，如
root@dev:/var/lib/docker/aufs# cat layers/c0cc0d951c469310d709a9de75fab1ae6ecc1b6bec69ff81796464879ea276ba
1a19b4ea83d007dd7fc1700f42e0fc9eac5eec93db2981890fb698b84cee6c41
01a1f4c8e4feaa9223ddb30e0c4bd0fec59acc5faf16480366c6e17a9ebdaaee
root@dev:/var/lib/docker/aufs# cat layers/1a19b4ea83d007dd7fc1700f42e0fc9eac5eec93db2981890fb698b84cee6c41
01a1f4c8e4feaa9223ddb30e0c4bd0fec59acc5faf16480366c6e17a9ebdaaee

### diff文件夹包含了diff的内容，文件夹名称是layer的cacheid
