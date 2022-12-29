---
layout: post
title: "cloudflare_wokers自定义域名反代"
date:   2022-12-06 15:51:13
tags: [转载,B站专栏文章,cloudflare_wokers反代,自定义域名,我看]
comments: true
author: Sun邦焱(uid：599959906)
toc: true
---

`转载按`：“适用于反代理加速访问目标网站，并通过自定义域方式解决cloudflare_wokers自带域名无法使用的问题。但是在添加DNS解析记录时，打开橙色云朵（全代理而非仅DNS）可能获得更好效果。”

<!-- more -->

## 准备

1.一个Cloudflare`账号`

2.一个已接入Cloudflare的`域名`（本实例使用sunbangyan.cn下的子域名1drv.sunbangyan.cn）

3.一个速度较快的`自选IP`（本实例使用104.16.127.166）

4.你的sharepoint组织名称（本实例为sunbangyan）

## 创建服务

首先，打开 dash.cloudflare.com，找到左侧的Workers，打开它。然后，轻点右方的“创建服务”。

创建页面显示什么不用管，直接划到下面，点击“创建服务”。

![创建服务页选项](https://ghproxy.com/https://raw.githubusercontent.com/hanlinniao/hanlinniao.github.io/master/images/2022-12-06-cloudflare_wokers%E5%8F%8D%E4%BB%A3/%E5%88%9B%E5%BB%BA%E9%A1%B5%E6%9C%8D%E5%8A%A1%E9%80%89%E9%A1%B9.jpg)

创建后打开右下方的“快速编辑”，复制下面的内容：

## js代码

```javascript
'use strict'

/**

* static files (404.html, sw.js, conf.js)
 */
const ASSET_URL = 'https://改成你的组织名称-my.sharepoint.com'

const JS_VER = 10
const MAX_RETRY = 1

/** @type {RequestInit} */
const PREFLIGHT_INIT = {
  status: 204,
  headers: new Headers({
    'access-control-allow-origin': '*',
    'access-control-allow-methods': 'GET,POST,PUT,PATCH,TRACE,DELETE,HEAD,OPTIONS',
    'access-control-max-age': '1728000',
  }),
}

/**

* @param {any} body
* @param {number} status
* @param {Object<string, string>} headers
 */
function makeRes(body, status = 200, headers = {}) {
  headers['--ver'] = JS_VER
  headers['access-control-allow-origin'] = '*'
  return new Response(body, {status, headers})
}

/**

* @param {string} urlStr
 */
function newUrl(urlStr) {
  try {
    return new URL(urlStr)
  } catch (err) {
    return null
  }
}

addEventListener('fetch', e => {
  const ret = fetchHandler(e)
    .catch(err => makeRes('cfworker error:\n' + err.stack, 502))
  e.respondWith(ret)
})

/**

* @param {FetchEvent} e
 */
async function fetchHandler(e) {
  const req = e.request
  const urlStr = req.url
  const urlObj = new URL(urlStr)
  const path = urlObj.href.substr(urlObj.origin.length)

  if (urlObj.protocol === 'http:') {
    urlObj.protocol = 'https:'
    return makeRes('', 301, {
      'strict-transport-security': 'max-age=99999999; includeSubDomains; preload',
      'location': urlObj.href,
    })
  }

  if (path.startsWith('/http/')) {
    return httpHandler(req, path.substr(6))
  }

  switch (path) {
  case '/http':
    return makeRes('请更新 cfworker 到最新版本!')
  case '/ws':
    return makeRes('not support', 400)
  case '/works':
    return makeRes('it works')
  default:
    // static files
    return fetch(ASSET_URL + path)
  }
}

/**

* @param {Request} req
* @param {string} pathname
 */
function httpHandler(req, pathname) {
  const reqHdrRaw = req.headers
  if (reqHdrRaw.has('x-jsproxy')) {
    return Response.error()
  }

  // preflight
  if (req.method === 'OPTIONS' &&
      reqHdrRaw.has('access-control-request-headers')
  ) {
    return new Response(null, PREFLIGHT_INIT)
  }

  let acehOld = false
  let rawSvr = ''
  let rawLen = ''
  let rawEtag = ''

  const reqHdrNew = new Headers(reqHdrRaw)
  reqHdrNew.set('x-jsproxy', '1')

  // 此处逻辑和 http-dec-req-hdr.lua 大致相同
  // <https://github.com/EtherDream/jsproxy/blob/master/lua/http-dec-req-hdr.lua>
  const refer = reqHdrNew.get('referer')
  const query = refer.substr(refer.indexOf('?') + 1)
  if (!query) {
    return makeRes('missing params', 403)
  }
  const param = new URLSearchParams(query)

  for (const [k, v] of Object.entries(param)) {
    if (k.substr(0, 2) === '--') {
      // 系统信息
      switch (k.substr(2)) {
      case 'aceh':
        acehOld = true
        break
      case 'raw-info':
        [rawSvr, rawLen, rawEtag] = v.split('|')
        break
      }
    } else {
      // 还原 HTTP 请求头
      if (v) {
        reqHdrNew.set(k, v)
      } else {
        reqHdrNew.delete(k)
      }
    }
  }
  if (!param.has('referer')) {
    reqHdrNew.delete('referer')
  }

  // cfworker 会把路径中的 `//` 合并成 `/`
  const urlStr = pathname.replace(/^(https?):\/+/, '$1://')
  const urlObj = newUrl(urlStr)
  if (!urlObj) {
    return makeRes('invalid proxy url: ' + urlStr, 403)
  }

  /** @type {RequestInit} */
  const reqInit = {
    method: req.method,
    headers: reqHdrNew,
    redirect: 'manual',
  }
  if (req.method === 'POST') {
    reqInit.body = req.body
  }
  return proxy(urlObj, reqInit, acehOld, rawLen, 0)
}

/**
 *

* @param {URL} urlObj
* @param {RequestInit} reqInit
* @param {number} retryTimes
 */
async function proxy(urlObj, reqInit, acehOld, rawLen, retryTimes) {
  const res = await fetch(urlObj.href, reqInit)
  const resHdrOld = res.headers
  const resHdrNew = new Headers(resHdrOld)

  let expose = '*'

  for (const [k, v] of resHdrOld.entries()) {
    if (k === 'access-control-allow-origin' ||
        k === 'access-control-expose-headers' ||
        k === 'location' ||
        k === 'set-cookie'
    ) {
      const x = '--' + k
      resHdrNew.set(x, v)
      if (acehOld) {
        expose = expose + ',' + x
      }
      resHdrNew.delete(k)
    }
    else if (acehOld &&
      k !== 'cache-control' &&
      k !== 'content-language' &&
      k !== 'content-type' &&
      k !== 'expires' &&
      k !== 'last-modified' &&
      k !== 'pragma'
    ) {
      expose = expose + ',' + k
    }
  }

  if (acehOld) {
    expose = expose + ',--s'
    resHdrNew.set('--t', '1')
  }

  // verify
  if (rawLen) {
    const newLen = resHdrOld.get('content-length') || ''
    const badLen = (rawLen !== newLen)

    if (badLen) {
      if (retryTimes < MAX_RETRY) {
        urlObj = await parseYtVideoRedir(urlObj, newLen, res)
        if (urlObj) {
          return proxy(urlObj, reqInit, acehOld, rawLen, retryTimes + 1)
        }
      }
      return makeRes(res.body, 400, {
        '--error': `bad len: ${newLen}, except: ${rawLen}`,
        'access-control-expose-headers': '--error',
      })
    }

    if (retryTimes > 1) {
      resHdrNew.set('--retry', retryTimes)
    }
  }

  let status = res.status

  resHdrNew.set('access-control-expose-headers', expose)
  resHdrNew.set('access-control-allow-origin', '*')
  resHdrNew.set('--s', status)
  resHdrNew.set('--ver', JS_VER)

  resHdrNew.delete('content-security-policy')
  resHdrNew.delete('content-security-policy-report-only')
  resHdrNew.delete('clear-site-data')

  if (status === 301 ||
      status === 302 ||
      status === 303 ||
      status === 307 ||
      status === 308
  ) {
    status = status + 10
  }

  return new Response(res.body, {
    status,
    headers: resHdrNew,
  })
}

/**

* @param {URL} urlObj
 */
function isYtUrl(urlObj) {
  return (
    urlObj.host.endsWith('.googlevideo.com') &&
    urlObj.pathname.startsWith('/videoplayback')
  )
}

/**

* @param {URL} urlObj
* @param {number} newLen
* @param {Response} res
 */
async function parseYtVideoRedir(urlObj, newLen, res) {
  if (newLen > 2000) {
    return null
  }
  if (!isYtUrl(urlObj)) {
    return null
  }
  try {
    const data = await res.text()
    urlObj = new URL(data)
  } catch (err) {
    return null
  }
  if (!isYtUrl(urlObj)) {
    return null
  }
  return urlObj
}
```

然后回到上面，修改ASSET_URL这一行,将'https://改成你的组织名称-my.sharepoint.com'修改为需要加速访问的目标网站'https://sunbangyan-my.sharepoint.com'

（本示例改为<https://sunbangyan-my.sharepoint.com>）

![js修改](https://ghproxy.com/https://raw.githubusercontent.com/hanlinniao/hanlinniao.github.io/master/images/2022-12-06-cloudflare_wokers%E5%8F%8D%E4%BB%A3/js%E4%BF%AE%E6%94%B9.jpg)

最后“保存并部署”，从左上角退出。

## 自定义域、路由及DNS解析记录添加

然后打开“触发器”，先在“自定义域”中，添加要作为反代服务的域名，等待3~5分钟后刷新页面，直到“证书”列显示为“有效”。

`（本实例使用sunbangyan.cn下的子域名1drv.sunbangyan.cn）`

![自定义域](https://ghproxy.com/https://raw.githubusercontent.com/hanlinniao/hanlinniao.github.io/master/images/2022-12-06-cloudflare_wokers%E5%8F%8D%E4%BB%A3/%E8%87%AA%E5%AE%9A%E4%B9%89%E5%9F%9F.jpg)

在下方点击“添加路由”，同样输入反代域名，但这里注意一点，最后一定要`加上/*`,区域选择对应的根域名即可。

![路由添加](https://ghproxy.com/https://raw.githubusercontent.com/hanlinniao/hanlinniao.github.io/master/images/2022-12-06-cloudflare_wokers%E5%8F%8D%E4%BB%A3/%E8%B7%AF%E7%94%B1%E6%B7%BB%E5%8A%A0.jpg)

接下来删除掉自定义域，如图所示。

![自定义域删除](https://ghproxy.com/https://raw.githubusercontent.com/hanlinniao/hanlinniao.github.io/master/images/2022-12-06-cloudflare_wokers%E5%8F%8D%E4%BB%A3/%E8%87%AA%E5%AE%9A%E4%B9%89%E5%9F%9F%E5%88%A0%E9%99%A4.jpg)

打开你的域名dns页，手动添加一条A记录解析，解析到你的自选IP。

并且点掉橙色云朵，改为“仅限DNS”。

![DNS解析记录添加](https://ghproxy.com/https://raw.githubusercontent.com/hanlinniao/hanlinniao.github.io/master/images/2022-12-06-cloudflare_wokers%E5%8F%8D%E4%BB%A3/DNS%E8%A7%A3%E6%9E%90%E8%AE%B0%E5%BD%95%E6%B7%BB%E5%8A%A0.jpg)

保存你的解析，至此，完成！

## 注意事项

自选IP需要自己找，文中的示例不一定一直可用。

参照这个项目 <https://github.com/XIU2/CloudflareSpeedTest/>
