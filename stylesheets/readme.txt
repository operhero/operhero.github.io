查看样式：rougify help style
下载样式：rougify style monokai.sublime > syntax_monokai.css
修改样式：_includes/head.html 添加 <link href='/stylesheets/syntax_monokai.css' rel='stylesheet' type='text/css' />
特别注意：pre code样式会被覆盖，深色背景不会生效。修改syntax_monokai.css，找到background-color的样式定义，添加 , pre code