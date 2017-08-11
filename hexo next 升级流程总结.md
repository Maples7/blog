成功升级，目前还没发现什么问题。
解决冲突还是需要点时间的，说一下我的流程：

1. 把自己本地的 theme/next 文件夹拷贝到一个新目录；
2. 在这个目录 `git init` -> `git add .` -> `git commit -m 'init'`；
3. `git remote add downstream git@github.com:iissnan/hexo-theme-next.git`；
4. `git fetch downstream v5.1.0`；
5. `git merge --no-ff FETCH_HEAD --allow-unrelated-histories`；
6. Shell 会输出哪些文件有冲突，耐心的一个文件一个文件的解决冲突（主要是自己对原主题有改动的地方要格外小心取舍）；
7. 解决完冲突之后可以直接把这个新建的目录下的所有文件拷贝到原博客项目目录下的 theme/next 下（除了 .git 文件夹）；
8. `hexo s -d` 查看 localhost:4000 具体页面上有哪些变动（明显的比如自定义页面在顶部比原来多了标题）；
9. 没有问题即可 `hexo clean && hexo g -d` 了；
10. 之后对于源代码该备份就备份、该 push 就 push。
