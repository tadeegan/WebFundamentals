project_path: /web/_project.yaml
book_path: /web/fundamentals/_book.yaml
description: 通过网络获取内容既缓慢，成本又高：大的响应需要在客户端和服务器之间进行多次往返通信，这拖延了浏览器可以使用和处理内容的时间，同时也增加了访问者的数据成本。因此，缓存和重用以前获取的资源的能力成为优化性能很关键的一个方面。

{# wf_review_required #}
{# wf_updated_on: 2014-01-04 #}
{# wf_published_on: 2013-12-31 #}

# HTTP 缓存 {: .page-title }

{% include "_shared/contributors/ilyagrigorik.html" %}



通过网络获取内容既缓慢，成本又高：大的响应需要在客户端和服务器之间进行多次往返通信，这拖延了浏览器可以使用和处理内容的时间，同时也增加了访问者的数据成本。因此，缓存和重用以前获取的资源的能力成为优化性能很关键的一个方面。



好消息是每个浏览器都实现了 HTTP 缓存！ 我们所要做的就是，确保每个服务器响应都提供正确的 HTTP 头指令，以指导浏览器何时可以缓存响应以及可以缓存多久。

<!-- TODO: Verify note type! -->
Note: 如果在应用中使用 Webview 来获取和显示网页内容，可能需要提供额外的配置标志，以确保启用了 HTTP 缓存，并根据用途设置了合理的缓存大小，同时，确保缓存持久化。查看平台文档并确认您的设置！

<img src="images/http-request.png" class="center" alt="HTTP 请求">

服务器在返回响应时，还会发出一组 HTTP 头，用来描述内容类型、长度、缓存指令、验证令牌等。例如，在上图的交互中，服务器返回了一个 1024 字节的响应，指导客户端缓存响应长达 120 秒，并提供验证令牌（`x234dff`），在响应过期之后，可以用来验证资源是否被修改。


## 使用 ETag 验证缓存的响应

## TL;DR {: .hide-from-toc }
- 服务器通过 ETag HTTP 头传递验证令牌
- 通过验证令牌可以进行高效的资源更新检查：如果资源未更改，则不会传输任何数据。


让我们假设在首次获取资源 120 秒之后，浏览器又对该资源发起了新请求。首先，浏览器会检查本地缓存并找到之前的响应，不幸的是，这个响应现在已经'过期'，无法在使用。此时，浏览器也可以直接发出新请求，获取新的完整响应，但是这样做效率较低，因为如果资源未被更改过，我们就没有理由再去下载与缓存中已有的完全相同的字节。

这就是 ETag 头中指定的验证令牌所要解决的问题：服务器会生成并返回一个随机令牌，通常是文件内容的哈希值或者某个其他指纹码。客户端不必了解指纹码是如何生成的，只需要在下一个请求中将其发送给服务器：如果指纹码仍然一致，说明资源未被修改，我们就可以跳过下载。

<img src="images/http-cache-control.png" class="center" alt="HTTP Cache-Control 示例">

在上面的例子中，客户端自动在`If-None-Match`HTTP 请求头中提供 ETag 令牌，服务器针对当前的资源检查令牌，如果未被修改过，则返回`304 Not Modified`响应，告诉浏览器缓存中的响应未被修改过，可以再延用 120 秒。注意，我们不必再次下载响应 - 这节约了时间和带宽。

作为网络开发人员，您如何利用高效的重新验证？ 浏览器代替我们完成了所有的工作：自动检测是否已指定了验证令牌，并会将验证令牌附加到发出的请求上，根据从服务器收到的响应，在必要时更新缓存时间戳。**实际上，我们唯一要做的就是确保服务器提供必要的 ETag 令牌：查看服务器文档中是否有必要的配置标志。**

<!-- TODO: Verify note type! -->
Note: 提示：HTML5 Boilerplate 项目包含了所有最流行的服务器的<a href='https://github.com/h5bp/server-configs'>配置文件样例</a>，并且为每个配置标志和设置都提供了详细的注释：在列表中找到您喜欢的服务器，查找适合的设置，然后复制/确认您的服务器配置了推荐的设置。


## Cache-Control

## TL;DR {: .hide-from-toc }
- 每个资源都可以通过 Cache-Control HTTP 头来定义自己的缓存策略
- Cache-Control 指令控制谁在什么条件下可以缓存响应以及可以缓存多久


最好的请求是不必与服务器进行通信的请求：通过响应的本地副本，我们可以避免所有的网络延迟以及数据传输的数据成本。为此，HTTP 规范允许服务器返回 [一系列不同的 Cache-Control 指令](http://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.9)，控制浏览器或者其他中继缓存如何缓存某个响应以及缓存多长时间。

<!-- TODO: Verify note type! -->
Note: Cache-Control 头在 HTTP/1.1 规范中定义，取代了之前用来定义响应缓存策略的头（例如 Expires）。当前的所有浏览器都支持 Cache-Control，因此，使用它就够了。

<img src="images/http-cache-control-highlight.png" class="center" alt="HTTP Cache-Control 示例">

### `no-cache` 和 `no-store`

`no-cache`表示必须先与服务器确认返回的响应是否被更改，然后才能使用该响应来满足后续对同一个网址的请求。因此，如果存在合适的验证令牌 (ETag)，no-cache 会发起往返通信来验证缓存的响应，如果资源未被更改，可以避免下载。

相比之下，`no-store`更加简单，直接禁止浏览器和所有中继缓存存储返回的任何版本的响应 - 例如：一个包含个人隐私数据或银行数据的响应。每次用户请求该资源时，都会向服务器发送一个请求，每次都会下载完整的响应。

### `public`和 `private`

如果响应被标记为`public`，即使有关联的 HTTP 认证，甚至响应状态码无法正常缓存，响应也可以被缓存。大多数情况下，`public`不是必须的，因为明确的缓存信息（例如`max-age`）已表示
响应可以被缓存。

相比之下，浏览器可以缓存`private`响应，但是通常只为单个用户缓存，因此，不允许任何中继缓存对其进行缓存 - 例如，用户浏览器可以缓存包含用户私人信息的 HTML 网页，但是 CDN 不能缓存。

### `max-age`

该指令指定从当前请求开始，允许获取的响应被重用的最长时间（单位为秒） - 例如：`max-age=60`表示响应可以再缓存和重用 60 秒。

## 定义最优 Cache-Control 策略

<img src="images/http-cache-decision-tree.png" class="center" alt="缓存决策树">

按照上面的决策树来确定您的应用使用的特定资源或一组资源的最优缓存策略。理想情况下，目标应该是在客户端上缓存尽可能多的响应、缓存尽可能长的时间，并且为每个响应提供验证令牌，以便进行高效的重新验证。

<table class="mdl-data-table mdl-js-data-table">
<thead>
  <tr>
    <th width="30%">Cache-Control 指令</th>
    <th>说明</th>
  </tr>
</thead>
<tbody>
<tr>
  <td data-th="cache-control">max-age=86400</td>
  <td data-th="说明">浏览器和任何中继缓存均可以将响应（如果是`public`的）缓存长达一天（60 秒 x 60 分 x 24 小时）</td>
</tr>
<tr>
  <td data-th="cache-control">private, max-age=600</td>
  <td data-th="说明">客户端浏览器只能将响应缓存最长 10 分钟（60 秒 x 10 分）</td>
</tr>
<tr>
  <td data-th="cache-control">no-store</td>
  <td data-th="说明">不允许缓存响应，每个请求必须获取完整的响应。</td>
</tr>
</tbody>
</table>

根据 HTTP Archive，在排名最高的 300,000 个网站中（Alexa 排名），[所有下载的响应中，几乎有半数可以由浏览器进行缓存](http://httparchive.org/trends.php#maxage0)，对于重复性网页浏览和访问来说，这是一个巨大的节省！ 当然，这并不意味着特定的应用会有 50% 的资源可以被缓存：有些网站可以缓存 90% 以上的资源， 而有些网站有许多私密的或者时间要求苛刻的数据，根本无法被缓存。

**审查您的网页，确定哪些资源可以被缓存，并确保可以返回正确的 Cache-Control 和 ETag 头。**


## 废弃和更新已缓存的响应

## TL;DR {: .hide-from-toc }
- 在资源"过期"之前，将一直使用本地缓存的响应
- 通过将文件内容指纹码嵌入网址，我们可以强制客户端更新到新版的响应
- 为了获得最佳性能，每个应用需要定义自己的缓存层级


浏览器发出的所有 HTTP 请求会首先被路由到浏览器的缓存，以查看是否缓存了可以用于实现请求的有效响应。如果有匹配的响应，会直接从缓存中读取响应，这样就避免了网络延迟以及传输产生的数据成本。**然而，如果我们希望更新或废弃已缓存的响应，该怎么办？**

例如，假设我们已经告诉访问者某个 CSS 样式表缓存长达 24 小时 (max-age=86400)，但是设计人员刚刚提交了一个更新，我们希望所有用户都能使用。我们该如何通知所有访问者缓存的 CSS 副本已过时，需要更新缓存？ 这是一个欺骗性的问题 - 实际上，至少在不更改资源网址的情况下，我们做不到。

一旦浏览器缓存了响应，在过期以前，将一直使用缓存的版本，这是由 max-age 或者 expires 指定的，或者直到因为某些原因从缓存中删除，例如用户清除了浏览器缓存。因此，在构建网页时，不同的用户可能使用的是文件的不同版本；刚获取该资源的用户将使用新版本，而缓存过之前副本（但是依然有效）的用户将继续使用旧版本的响应。

**所以，我们如何才能鱼和熊掌兼得：客户端缓存和快速更新？** 很简单，在资源内容更改时，我们可以更改资源的网址，强制用户下载新响应。通常情况下，可以通过在文件名中嵌入文件的指纹码（或版本号）来实现 - 例如 style.**x234dff**.css。

<img src="images/http-cache-hierarchy.png" class="center" alt="Cache 层级">

因为能够定义每个资源的缓存策略，所以，我们可以定义'缓存层级'，这样，不但可以控制每个响应的缓存时间，还可以控制访问者看到新版本的速度。例如，我们一起分析一下上面的例子：

* HTML 被标记成`no-cache`，这意味着浏览器在每次请求时都会重新验证文档，如果内容更改，会获取最新版本。同时，在 HTML 标记中，我们在 CSS 和 JavaScript 资源的网址中嵌入指纹码：如果这些文件的内容更改，网页的 HTML 也会随之更改，并将下载 HTML 响应的新副本。
* 允许浏览器和中继缓存（例如 CDN）缓存 CSS，过期时间设置为 1 年。注意，我们可以放心地使用 1 年的'远期过期'，因为我们在文件名中嵌入了文件指纹码：如果 CSS 更新，网址也会随之更改。
* JavaScript 过期时间也设置为 1 年，但是被标记为 private，也许是因为包含了 CDN 不应缓存的一些用户私人数据。
* 缓存图片时不包含版本或唯一指纹码，过期时间设置为 1 天。

组合使用 ETag、Cache-Control 和唯一网址，我们可以提供最佳的方案：较长的过期时间，控制可以缓存响应的位置，以及按需更新。

## 缓存检查表

不存在最佳的缓存策略。根据您的通信模式、提供的数据类型以及应用特定的数据更新要求，必须定义和配置每个资源最适合的设置以及整体的'缓存层级'。

在定义缓存策略时，要记住下列技巧和方法：

1. **使用一致的网址：**如果您在不同的网址上提供相同的内容，将会多次获取和存储该内容。提示：注意，[网址区分大小写](http://www.w3.org/TR/WD-html40-970708/htmlweb.html)！
2. **确保服务器提供验证令牌 (ETag)：**通过验证令牌，如果服务器上的资源未被更改，就不必传输相同的字节。
3. **确定中继缓存可以缓存哪些资源：**对所有用户的响应完全相同的资源很适合由 CDN 或其他中继缓存进行缓存。
4. **确定每个资源的最优缓存周期：**不同的资源可能有不同的更新要求。审查并确定每个资源适合的 max-age。
5. **确定网站的最佳缓存层级：**对 HTML 文档组合使用包含内容指纹码的资源网址以及短时间或 no-cache 的生命周期，可以控制客户端获取更新的速度。
6. **搅动最小化：**有些资源的更新比其他资源频繁。如果资源的特定部分（例如 JavaScript 函数或一组 CSS 样式）会经常更新，应考虑将其代码作为单独的文件提供。这样，每次获取更新时，剩余内容（例如不会频繁更新的库代码）可以从缓存中获取，确保下载的内容量最少。



