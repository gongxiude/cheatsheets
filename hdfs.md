
## 说明
```
➜ tree -L 3 o2o-ops-script
o2o-ops-script
├── ansible.cfg         #ansible配置文件
├── build.sh            #jenkins构建统一入口文件
├── deploy-front.yml    #发布前端ansible playbook
├── deploy-java.yml     #发布java程序ansible playbook
├── hosts               #ansible 主机资源文件， 如果增加或删除主机修改此文件
├── readme.md
├── scripts              
│   ├── public          #公共类脚本存放目录， 如当前的autoTest.sh或nginx日志切割脚本等
│   │   ├── autoTest.sh
│   │   ├── deploy-callback.sh
│   │   └── testCopyProd.sh
│   └── startup         #程序启动脚本， 按照环境维护
│       ├── prod
│       ├── test001
│       ├── test002
│       ├── test003
│       ├── test004
│       └── test005
└── scripts.yml

9 directories, 12 files
```