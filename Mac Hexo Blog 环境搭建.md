# Mac Hexo Blog 环境搭建


---


### 一、前置条件

1. 安装Node.js

    官网地址： [Node.js][1]
    
    安装Node过程中可能存在Permission问题，在安装命令前加sudo就好。
    
    ![此处输入图片的描述][2]
    
2. 安装Git
    
    此处输入代码Mac下默认安装了Git所以忽略。

3. GitHub账号SSH Key注册

    *  在github上建议一个名为『your_user_name.github.com』的仓库
    
    *  添加SSH公钥到github SSH Key
    
    *  本地git配置
    
            git config --global user.email "your email address"
            
            git config --global user.name "your github name"

    *   生成密钥
    
            ssh-keygen -t rsa -C "your email address"

        拷贝生成的公钥和私钥文件至.ssh的目录下，如果没有权限，需要chmod
    

 


----------


   
### 二、安装hexo

1. 安装hexo
    如果安装中遇到Permision问题，加sudo

        npm install -g hexo
    

2. 初始化

        hexo init <folder>

3. 启动调试

        hexo server


----------


### 三、个性化

1.  修改博客标题
    修改根目录下的_config.yml文件内容，例如blog title等。
2.  更换主题
    在theme目录下clone一份主题代码。
`$ git clone https://github.com/ppoffice/hexo-theme-hueman.git themes/hueman`

3.  更换摘要图片


----------


### 四、采坑

1. hexo deploy 没反应
    文章写好了，想deploy到github上时，其他命令都执行正常，hexo deploy后无任何输出，参考网上的各种所谓教程，都有输出，而均未提起怎样才能正常deploy。最开始还以为是github上仓库创建不对。后来才发现罪魁祸首是_config.yml的配置。




        # Deployment
        ## Docs: https://hexo.io/docs/deployment.html
        deploy:
        type: git
        repo: git@github.com:protoss-zealot/protoss-zealot.github.com.git
        branch: master
**FBI Warning: type、repo、branch后面需要有一个空格！**

2. 更换主题后文章打不开，提示 github 404

    有可能是你的文章编译未及时更新到github。删掉本地hexo目录下的.deploy_git文件，再次deploy就好了。
    
        $ hexo clean
        $ rm -rf .deploy_git

----------


### References

 1. [https://hexo.io/zh-cn/docs/configuration.html][3]
 2. [http://ibruce.info/2013/11/22/hexo-your-blog/][4]
 3. [https://github.com/hexojs/hexo/issues/1154][5]
 4. [如何管理多个SSH key][6]


  [1]: https://nodejs.org/en/#download
  [2]: https://s1.ax2x.com/2017/12/19/zW4z3.jpg
  [3]: https://hexo.io/zh-cn/docs/configuration.html
  [4]: http://ibruce.info/2013/11/22/hexo-your-blog/
  [5]: https://github.com/hexojs/hexo/issues/1154
  [6]: http://blog.csdn.net/wwmusic/article/details/51027458