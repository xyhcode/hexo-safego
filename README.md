## hexo-safego

[NPM发布地址](https://www.npmjs.com/package/hexo-safego)

`hexo-safego` 是一个改进版的 Hexo 插件，用于处理外部链接以增强博客的安全性。该插件基于 [`hexo-external-link`](https://github.com/hvnobug/hexo-external-link) 二次开发，但其实现方式不同：原版插件通过注入 JS ，在打开网站时才处理外部链接，而 `hexo-safego` 则在生成静态页面时直接替换外部链接，并且提供了很多人性化的配置。

**注意：该插件由于刚发版,再加上近几天比较忙，暂时并不完善，仍在一直测试阶段，配置不适配更新比较频繁，如果无法运行，可以选择同类替代产品，如：[安全跳转页面重制版](https://blog.qyliu.top/posts/9efc1657/)，等测试期结束后，我将会推出兼容性良好的版本，如果感兴趣可以star以关注后续更新！**

### 主要特性

- **外部链接跳转**：将外部链接替换为自定义的跳转页面，增加安全性。
- **灵活配置**：支持多个容器 ID、白名单域名和生效页面路径的配置。
- **Base64 编码**：可选的 Base64 编码功能。
- **调试模式**：调试模式输出详细信息，便于开发和调试。
- **自定义页面**：本插件自定义页面简单明了，完全可以进行自定义以达到自己的要求，目前初步支持暗黑模式。
- **自定义化**：目前支持设置标题，副标题，头像，暗黑模式时间等，不过不稳定请谨慎更新，添加了一些参数

### 安装
 
在使用该插件之前，需要先安装 `cheerio`，hexo一般有这个插件，可以在 `node_module` 查看，如果没有，执行：

```bash
npm install cheerio --save
```

然后安装 `hexo-safego` 插件：

```bash
npm install hexo-safego --save
```

### Hexo-Safego 插件配置更新

在 Hexo 的配置文件 `_config.yml` 中添加或更新以下配置（比较繁杂，正在测试，后面会去掉不使用的内容）：

```yaml
hexo_safego:
  enable: true  # 是否启用 hexo-safego 插件
  enable_base64_encode: true  # 是否启用 Base64 编码链接
  url_param_name: 'u'  # URL 参数名，用于生成跳转链接
  html_file_name: 'go.html'  # 跳转页面文件名
  target_blank: true  # 是否在跳转链接中添加 target="_blank"
  link_rel: 'external nofollow noopener noreferrer'  # 跳转链接的 rel 属性
  ignore_attrs:  # 需要忽略的链接属性列表
    - 'data-fancybox'
    - 'ignore-external-link'
  container_ids:  # 容器 ID 列表，如果为空则匹配整个 body
    - 'article-container'
  domain_whitelist:  # 域名白名单列表，包含白名单中的域名的链接将被忽略
  apply_pages:  # 生效页面路径列表，只有在这些页面上才会对链接进行处理
    - '/posts/'
  debug: false  # 是否启用调试模式，开启后会输出详细的调试信息
  avatar: "https://pic.imgdb.cn/item/6633cb0b0ea9cb1403cc54a4.webp"  # 头像图片链接
  title: "清羽飞扬"  # 标题
  subtitle: "安全中心"  # 副标题
  darkmode:  # 夜间模式配置
    enable: true
    start: 18  # 夜间模式开始时间
    end: 6  # 夜间模式结束时间
```

### 插件主要代码实现

以下是 `hexo-safego` 插件的关键代码实现，主要通过 `cheerio` 来解析和处理 HTML 内容：

```javascript
const cheerio = require('cheerio');

const config = hexo.config.hexo_safego = Object.assign({
    enable: false,
    enable_base64_encode: true,
    url_param_name: 'u',
    html_file_name: 'go.html',
    target_blank: true,
    domain: '',
    safety_chain: false,
    link_rel: 'external nofollow noopener noreferrer',
    ignore_attrs: [],
    container_ids: ['article-container'],  // 容器ID列表，如果为空则匹配body
    domain_whitelist: [],  // 域名白名单列表
    apply_pages: ['/posts/'],  // 生效页面路径列表
    debug: false,  // 调试参数，默认为false
    avatar: "https://pic.imgdb.cn/item/6633cb0b0ea9cb1403cc54a4.webp",
    title: "清羽飞扬",
    subtitle: "安全中心",
    darkmode: {
        enable: true,
        start: 18,
        end: 6
    }
}, hexo.config.hexo_safego);

const default_ignore_attrs = ['data-fancybox', 'ignore-external-link'];
const ignore_attrs = Array.from(new Set(default_ignore_attrs.concat(config.ignore_attrs)));
const root = hexo.config.root || '/';

if (config.enable) {
    hexo.extend.filter.register('after_render:html', function (htmlContent, data) {
        const $ = cheerio.load(htmlContent);

        if (config.debug) {
            console.log("Processing links within specified containers:", config.container_ids);
        }

        const currentPath = '/' + data.path;
        if (config.debug) {
            console.log("Current page path:", currentPath);
        }

        const isPathInApplyPages = config.apply_pages.some(page => {
            if (page === '/') {
                return true;  // 如果设置为 '/'，则对所有页面生效
            }
            return currentPath.startsWith(page);
        });

        if (!isPathInApplyPages) {
            if (config.debug) {
                console.log("Current page path is not in the apply_pages list, skipping link processing.");
            }
            return htmlContent;
        }

        const containers = config.container_ids.length ? config.container_ids : ['body'];

        containers.forEach(id => {
            const selector = id === 'body' ? 'body a' : `#${id} a`;
            $(selector).each(function() {
                const $this = $(this);
                const href = $this.attr('href');

                if (!href) return;

                const hasAttr = ignore_attrs.some(attr => $this.attr(attr) !== undefined);
                if (hasAttr) {
                    if (config.debug) {
                        console.log("Link ignored due to attribute match:", href);
                    }
                    return;
                }

                const isLinkInWhitelist = config.domain_whitelist.some(whitelistDomain => href.includes(whitelistDomain));
                if (isLinkInWhitelist) {
                    if (config.debug) {
                        console.log("Link in whitelist, ignoring link:", href);
                    }
                    return;
                }

                if (href.match('^((http|https|thunder|qqdl|ed2k|Flashget|qbrowser|ftp|rtsp|mms)://)')) {
                    const strs = href.split('/');
                    if (strs.length >= 3) {
                        const host = strs[2];
                        if (host !== config.domain) {
                            if (config.debug) {
                                console.log("External link detected:", href);
                            }
                            const encodedHref = config.enable_base64_encode ? Buffer.from(href).toString('base64') : href;
                            const newHref = `${root}${config.html_file_name}?${config.url_param_name}=${encodedHref}`;
                            $this.attr('href', newHref).attr('rel', config.link_rel);
                            if (config.target_blank) {
                                $this.attr('target', '_blank');
                            }
                        }
                    }
                }
            });
        });

        return $.html();
    });
    hexo.extend.generator.register('external_link', require('./lib/generator'));
}
```

### 跳转页面生成

插件会根据配置自动生成一个 `go.html` 文件，作为跳转页面。以下是生成跳转页面的代码：

```javascript
'use strict';

const nunjucks = require('nunjucks');
const env = new nunjucks.Environment();
const pathFn = require('path');
const fs = require('fs');

env.addFilter('uriencode', str => {
    return encodeURI(str);
});

env.addFilter('noControlChars', str => {
    return str.replace(/[\x00-\x1F\x7F]/g, '');
});

const goTmplSrc = pathFn.join(__dirname, '../go.html');
const template = nunjucks.compile(fs.readFileSync(goTmplSrc, 'utf8'), env);

module.exports = function (locals) {
    const config = this.config;
    const fel_config = config.hexo_safego;
    const html = template.render({
        url_param_name: fel_config.url_param_name,
        enable_base64_encode: fel_config.enable_base64_encode,
        domain: fel_config.domain,
        safety_chain: fel_config.safety_chain,
        avatar: fel_config.avatar,
        title: fel_config.title,
        subtitle: fel_config.subtitle,
        darkmode_enable: fel_config.darkmode.enable,
        darkmode_start: fel_config.darkmode.start,
        darkmode_end: fel_config.darkmode.end,
    });
    return {
        path: fel_config.html_file_name,
        data: html
    };
};
```

### 总结

`hexo-safego` 提供了一种灵活且安全的方式来处理外部链接，通过在生成页面时直接替换外部链接，提高了网站的安全性和可控性。配置简单易用，适用于需要增强外部链接安全性的 Hexo 博客用户。