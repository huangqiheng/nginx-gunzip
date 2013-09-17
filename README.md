nginx-gunzip
============

在nginx作为正向代理的时候，原官方的gunzip模块，在客户端发送gzip的header时将不会生效，这让在正向代理中过滤内容的substitute模块无法工作。参考淘宝姚伟斌的补丁，打包了一个方便自己使用的。
