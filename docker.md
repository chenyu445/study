<!--
 * @Description: 
 * @Author: Damon.chen
 * @LastEditors: Damon.chen
 * @Date: 2019-04-15 14:52:28
 * @LastEditTime: 2019-04-15 23:16:11
 -->
#删除tag
docker rmi  index-dev.qiniu.io/cs-kirk/nginx:latest
docker rmi index-dev.qiniu.io/cs-kirk/nginx@sha256:c7c1149150a8f7536bd19b70ea34748bf9dfbc93e5dee677f11362c06b12735d
＃删除曾经启动过的容器
docker rm containerid
#删除没有tag的镜像
docker rmi e9d3f7300f03
批量删除本地images
 docker rmi $(docker images |awk '{if($2=="<none>")  print $3}')