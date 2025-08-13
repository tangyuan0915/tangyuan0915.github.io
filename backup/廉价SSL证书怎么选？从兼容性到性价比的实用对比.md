提到免费SSL证书，很多人第一反应是Let’s Encrypt——它确实是推动HTTPS普及的功臣，支持ECDSA算法和泛域名，门槛低到几乎零成本。但90天的有效期总让人头疼：如果系统里有不少没法自动续期的设备（比如老路由器、嵌入式终端），手动管理的成本会直线上升。所以今天就来聊聊几个性价比高的付费选项：Sectigo、RapidSSL和AlphaSSL，看看它们在兼容性、安全性和价格上到底有啥差别。


## Sectigo：老牌厂商的性价比之选

Sectigo其实就是以前的Comodo（后来拆分后改名），在廉价SSL证书里算是老面孔了，中小网站用得特别多。它家的根证书阵容很扎实，有四个核心根证书：COMODO RSA、COMODO ECC、USERTrust RSA、USERTrust ECC，有效期从2010年（ECC根是2008年）一直到2038年，虽然不算最老牌，但胜在安全性新，完美支持ECDSA（包括RSA4096、ECC384加密，搭配SHA384哈希）。

不过老设备兼容性得注意：这些新根证书在太旧的系统（比如Windows XP、老款Linux）里可能不被信任。好在Sectigo早年收购过一个2000年启用的AddTrust根证书，可惜2020年就过期了；好在还有个2004年的AAA Certificate Services根证书，能用到2028年，老系统里兼容性很好。只要在服务器证书链里加个交叉签名的中级证书，新颁发的证书就能被老设备认出来，就是证书链会变长（多一个中级证书），稍微费点流量。

价格方面，现在GoGetSSL上的Sectigo泛域名证书很划算，3年期均价大概$35/年。有意思的是，GoGetSSL自己的中级证书（GoGetSSL RSA DV CA、GoGetSSL ECC DV CA）是Sectigo根证书签的，所以用GoGetSSL的证书和直接买Sectigo的兼容性没区别，性价比反而更高。


## RapidSSL：大厂背书的稳定选项

RapidSSL是DigiCert旗下的亲民品牌，背靠DigiCert的根证书体系，好处是兼容性不用太操心。不过它的根证书和中级证书只支持RSA算法，对ECDSA有需求的话就得绕道了。

名气大的好处是转售渠道多，价格也透明。现在SSL2BUY上的2年期RapidSSL泛域名证书，均价约$75/年；如果买3年期，CheapSSLsecurity能谈到$70/年左右，长期用的话性价比会更高一点。适合那些看重品牌稳定性，对算法没特殊要求的场景。


## AlphaSSL：小众但便宜的黑马

AlphaSSL是GlobalSign旗下的廉价线，用的是GlobalSign的根证书，同样只支持RSA。它的名气比前两个小很多，转售平台也少，但价格很有吸引力——目前SSL2BUY的2年期AlphaSSL泛域名证书，大概$42/年，比RapidSSL便宜不少，预算有限又不需要ECDSA的话可以留意。


## 总结：按需求挑，不花冤枉钱

简单说，如果你的设备需要全链路ECDSA支持（比如对性能敏感的场景），GoGetSSL的Sectigo证书是目前最便宜的选择，就是老设备可能得处理四级证书链；如果不执着于ECDSA，RapidSSL和AlphaSSL更省心——三级证书链足够，老终端兼容性也好，RapidSSL名气大些，AlphaSSL则胜在性价比。

顺便说句证书链的小知识：现在的SSL证书几乎都是由中级证书签发的（根证书太敏感，不会直接签用户证书），所以三级证书链（根→中级→用户证书）基本是最短的。多一级中级证书的话，服务器会多传约2KB的数据，客户端验证时也得多查一次证书吊销状态，对现代设备影响不大，但老终端可能会慢一点点。

最后，如果想确认某个根证书在主流浏览器里的信任情况，推荐看看“[CAs trustworthy in browsers db](https://labs.vjirovsky.cz/cas-info/)”，里面能查到各根证书的信任范围，选的时候心里更有底。