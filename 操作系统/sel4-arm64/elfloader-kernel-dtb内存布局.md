# CPIO格式
elfloader使用cpio工具将kernel、dtb(optional)、app(root server)打包成cpio格式，并将该cpio包插入到其镜像中，结合链接脚本符号，进而在elfloader中找到cpio包，进一步地，对cpio进行解析，方便后续加载与boot

