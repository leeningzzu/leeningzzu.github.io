---
title: serverless+travis-ci部署hexo博客
catalog: true
toc_nav_num: true
subtitle: github博客搭建
tags:
  - blog
  - hexo
categories:
  - Software and Skillings
  - Skillings
abbrlink: fbf2107d
date: 2019-03-01 03:09:06
header-img:
---

## hexo + github构建
1. 下载部署git、hexo，主要参考[韦阳的博客](https://zhuanlan.zhihu.com/p/35668237)
2. 采用hux主题并进行优化
- [x] 改善的SEO与title同步的情况
- [x] 采用valine评论系统，删除多说及discuss
- [x] 增加分享分能，文章标题上方添加tags及categories
- [ ] 添加搜索功能
- [ ] 将tags与categories合并为一页显示
- [ ] 实现文章监听滚动功能

注意事项： node的最新版本，以及hux博客的优化版本如[YuankunLi](https://github.com/CatherineLiyuankun/Hexo-theme-zilan) and [huweihuang](https://github.com/huweihuang/hexo-theme-huweihuang)。其中mathjax公式插件参考[Hexo使用mathjax渲染公式](https://wudaijun.com/2017/12/hexo-with-mathjax/)

## travis-ci
主要参考[Travis CI自动部署Hexo博客到GitHub](https://blog.qizhenjun.com/75a7da42/)及[使用Travis CI自动部署Hexo博客](https://www.itfanr.cc/2017/08/09/using-travis-ci-automatic-deploy-hexo-blogs/)<br />注意事项：创建一个新分支**hexo**用于存放未进行构建的源码，而**master**分支为作为构建后输出目录，**travis.yml**跟踪监听**hexo**分支，并推送github的**master**分支

**git推送**
```R 
git add .
git commit -m "new blog"
git push 
```

##  采用腾讯云serverless +语雀进行自动化部署 
注意事项：
1. win10 不必要创建travis-ci私密密钥，cmd内已自带curl函数，administrator打开cmd可采用curl获取相应仓库id

```bash
curl -H "Travis-API-Version: 3" -H "User-Agent: API Explorer" \ -H "Authorization: token <your_token>" \ https://api.travis-ci.org/owner/<your_name>/repos
```
2. 因为在travis-ci部署中创建hexo分支进行监听，serverless函数中监听branch应为hexo
**Serverless**
```php
<?php
function main_handler($event, $context) {
    // 解析语雀post的数据
    $update_title = '';
    if($event->body){
        $yuque_data= json_decode($event->body);
        $update_title .= $yuque_data->data->title;
    }
    // default params
    $repos = '';  // 你的仓库id 或 slug
    $token = ''; // 你的登录token
    $message = date("Y/m/d").':yuque update:'.$update_title;
    $branch = 'hexo';
    // post params
    $queryString = $event->queryString;
    $q_token = $queryString->token ? $queryString->token : $token;
    $q_repos = $queryString->repos ? $queryString->repos : $repos;
    $q_message = $queryString->message ? $queryString->message : $message;
    $q_branch = $queryString->branch ? $queryString->branch : 'hexo';
    echo($q_token);
    echo('===');
    echo ($q_repos);
    echo ('===');
    echo ($q_message);
    echo ('===');
    echo ($q_branch);
    echo ('===');
    //request travis ci
    $res_info = triggerTravisCI($q_repos, $q_token, $q_message, $q_branch);

    $res_code = 0;
    $res_message = '未知';
    if($res_info['http_code']){
        $res_code = $res_info['http_code'];
        switch($res_info['http_code']){
            case 200:
            case 202:
                $res_message = 'success';
            break;
            default:
                $res_message = 'faild';
            break;
        }
    }
    $res = array(
        'status'=>$res_code,
        'message'=>$res_message
    );
    return $res;
}

/*
* @description  travis api , trigger a build
* @param $repos string 仓库ID、slug
* @param $token string 登录验证token
* @param $message string 触发信息
* @param $branch string 分支
* @return $info array 回包信息
*/
function triggerTravisCI ($repos, $token, $message='yuque update', $branch='hexo') {
    //初始化
    $curl = curl_init();
    //设置抓取的url
    curl_setopt($curl, CURLOPT_URL, 'https://api.travis-ci.org/repo/'.$repos.'/requests');
    //设置获取的信息以文件流的形式返回，而不是直接输出。
    curl_setopt($curl, CURLOPT_RETURNTRANSFER, 1);
    //设置post方式提交
    curl_setopt($curl, CURLOPT_CUSTOMREQUEST, "POST");
    //设置post数据
    $post_data = json_encode(array(
        "request"=> array(
            "message"=>$message,
            "branch"=>$branch
        )
    ));
    $header = array(
      'Content-Type: application/json',
      'Travis-API-Version: 3',
      'Authorization:token '.$token,
      'Content-Length:' . strlen($post_data)
    );
    curl_setopt($curl, CURLOPT_HTTPHEADER, $header);
    curl_setopt($curl, CURLOPT_POSTFIELDS, $post_data);
    //执行命令
    $data = curl_exec($curl);
    $info = curl_getinfo($curl);
    //关闭URL请求
    curl_close($curl);
    return $info;
}
?>
```
3.语雀webhook配置时

```basic
https://service-<…………>.ap-guangzhou.apigateway.myqcloud.com/release/main_handler?repos=<your travis's repo ID>&token=<your travis token>&message=语雀自动提交&branch=hexo
```

主要参考[Nero大神](https://segmentfault.com/a/1190000017797561)，[raylee](https://rayleeafar.github.io/2019/01/17/yuque/%E8%AF%A6%E7%BB%86Hexo%E5%8D%9A%E5%AE%A2%E6%90%AD%E5%BB%BA%EF%BC%9A%E4%BA%91%E7%AB%AF%E5%86%99%E4%BD%9C+%E8%87%AA%E5%8A%A8%E6%9E%84%E5%BB%BA+%E8%87%AA%E5%8A%A8%E9%83%A8%E7%BD%B2/),以及[Zen](https://iszengmh.github.io/2019/01/27/yuque/%E8%AF%AD%E9%9B%80%E4%B9%8B%E8%AF%AD%E9%9B%80+serverless+travis%20CI+hexo+github%E6%90%AD%E5%BB%BA%E4%BA%91%E5%86%99%E4%BD%9C%E5%8D%9A%E5%AE%A2/)


## 语雀使用体会
1. 方便快捷，类似于word的可视化,自动同步
2. 支持markdown功能较少，front-matter实现不佳，尚不支持公式编辑，功能尚需完善
4. 目前以travis-ci 本地自动部署为主
