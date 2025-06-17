# test

```bash
领域	难度	故障注入命令（仅 1 行）	作用点 & 预期现象
认证 / 授权	初级	kubectl -n $NS patch deploy podinfo --type=json -p='[{"op":"add","path":"/spec/template/spec/serviceAccountName","value":"ghost-sa"}]'	Pod 创建失败 → FailedCreate serviceaccount "ghost-sa" not found。让学员理解 ServiceAccount ↔ Pod 的依赖。
中级	kubectl -n $NS patch deploy podinfo --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"PODINFO_BASIC_AUTH","value":"user:pwd"}}]'	开启 BasicAuth
github.com
；外部探针和 UI 无凭证即 401/403，学员需移除变量或在请求中加 Authorization 头。
高级	kubectl -n $NS patch deploy podinfo --type=json -p='[{"op":"add","path":"/spec/template/spec/automountServiceAccountToken","value":false}]'	JWT 签发端点 /token 连 GET /readyz 都失效（缺少 SA token）→ readiness 500；要么重开自动挂载，要么改用 ClusterIP 直连 Redis 等不需 JWT 的接口。
存储	初级	kubectl -n $NS patch deploy podinfo --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/-","value":{"name":"oops","mountPath":"/data"}}]'	加了 volumeMount 却 没声明 Volume → CreateContainerConfigError（Kube 事件：mount name not found）。
中级	kubectl -n $NS patch deploy podinfo --type=json -p='[{"op":"add","path":"/spec/template/spec/volumes/-","value":{"name":"ro","emptyDir":{}}},{"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/-","value":{"name":"ro","mountPath":"/data","readOnly":true}}]'	/data 变只读；留言板写磁盘 /store 时 500，日志 read-only file system。
高级	kubectl -n $NS patch deploy podinfo --type=json -p='[{"op":"add","path":"/spec/template/spec/volumes/-","value":{"name":"hp","hostPath":{"path":"/no/such/dir","type":"Directory"}}},{"op":"add","path":"/spec/template/spec/containers/0/volumeMounts/-","value":{"name":"hp","mountPath":"/data"}}]'	HostPath 指向不存在目录 → Pod卡在 Pending，事件 path "/no/such/dir" does not exist.
环境变量	初级	kubectl -n $NS patch deploy podinfo --type=json -p='[{"op":"replace","path":"/spec/template/spec/containers/0/env/0/value","value":"redis://wrong:6379"}]'	把 PODINFO_CACHE_SERVER 指向不存在主机 → /healthz 立刻 500（Redis ping fail）。
中级	kubectl -n $NS patch deploy podinfo --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"PODINFO_FAULTS_UNREADY","value":"true"}}]'	打开 故障注入（值来自 Helm faults.unready）
github.com
→ Readiness 随机 500/timeout。
高级	kubectl -n $NS patch deploy podinfo --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/env/-","value":{"name":"PODINFO_PORT","value":"9999"}}]'	Podinfo 在 9999 启动，但 Service/Ingress 仍指 9898 → 内部 OK，外部全部 503；需同步改 Service/Ingress 或变量回滚。
Deployment 本身	初级	kubectl -n $NS scale deploy podinfo --replicas=0	故障最直观：Ingress 503；学员只需把副本数调回。
中级	kubectl -n $NS patch deploy podinfo --type=json -p='[{"op":"replace","path":"/spec/template/spec/containers/0/image","value":"stefanprodan/podinfo:0.1.0"}]'	换到 超旧镜像，健康探针缺 Redis 支持 → readiness fail。学生需回滚 tag 或看 ImagePullBackOff。
高级	kubectl -n $NS patch deploy podinfo --type=json -p='[{"op":"add","path":"/spec/template/spec/containers/0/command","value":["/bin/sh","-c","exit 1"]}]'	强制容器立即退出 → CrashLoopBackOff。要去掉 command 或改成正确 Entrypoint。
Service	初级	kubectl -n $NS patch svc podinfo --type=json -p='[{"op":"replace","path":"/spec/selector/app\\.kubernetes\\.io~1name","value":"typo"}]'	Selector 不匹配 → Endpoints 空 → Ingress 503。
中级	kubectl -n $NS patch svc podinfo-redis --type=json -p='[{"op":"replace","path":"/spec/ports/0/targetPort","value":6381}]'	Service 端口对不上 Pod → /healthz 500。修复需还原 targetPort 或更新 Redis containerPort。
高级	kubectl -n $NS patch svc podinfo --type=json -p='[{"op":"replace","path":"/spec/type","value":"ExternalName"},{"op":"add","path":"/spec/externalName","value":"example.com"}]'	Service 变成 ExternalName → DNS 解到外网，Ingress 解析失败；学员需改回 ClusterIP 并重建 Endpoints。
