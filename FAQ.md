# Ubuntu max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]
使用root用户
1. 修改系统配置/etc/security/limits.conf
```shell
cat <<EOF>> /etc/security/limits.conf
${ELSTICSEARCH_USERNAME} hard nofile 65536
${ELSTICSEARCH_USERNAME} soft nofile 65536
EOF
```
2. 使用${ELSTICSEARCH_USERNAME} 重新登录
3. 检查修改结果
```shell
ulimit -n
65536
```

# max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
1.  修改系统配置/etc/sysctl.conf
```shell
cat <<EOF>> /etc/sysctl.conf
vm.max_map_count=262144
EOF
```
2. 查看修改结果
```shell
sysctl -p
vm.max_map_count = 262144
```
