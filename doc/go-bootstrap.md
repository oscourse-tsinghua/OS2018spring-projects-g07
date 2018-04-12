# GO bootstrap

需要执行以下步骤：

+ 在某一文件夹下执行 git clone https://github.com/golang/go.git, 并切换到release-branch.go1.4分支
+ 在分支中进入src文件夹，执行make.bash
+ 将环境变量GOROOT_BOOTSTRAP设为go项目根目录，即要求$GOROOT_BOOTSTRAP/bin/go存在

以上都完成后，才可以进入本工程。