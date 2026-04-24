`本机打完镜像之后，经过tag，然后push时，突然报错，内容是：Invalid image, fail to parse 'manifest.json'`
>这通常并不是docker build的问题，而是镜像格式与仓库不兼容。
>最常见原因是Docker/BuildKit版本过新，仓库不支持OCI Manifest V2. 如果使用了docker version25+
>验证方式：docker manifest inspect your-image:tag
>如果结果是："mediaType": "application/vnd.oci.image.manifest.v1+json"
>   或者是："manifests": [
  {
    "platform": { "architecture": "amd64", ... }
  }
]
>基本就可以确认是仓库不支持，修正方法：强制使用 classic docker v2 schema
```shell
DOCKER_BUILDKIT=0 docker build -t your-image:tag .
docker push your-image:tag
```
```shell
# 如果在使用buildx
docker buildx build \
  --platform linux/amd64 \
  --output=type=docker \
  -t your-image:tag .
docker push your-image:tag
# 关键点是--output=type=docker,不然的话默认就是oci
```