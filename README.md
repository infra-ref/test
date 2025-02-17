# test
如果你目前部署的 Grafana 没有启用持久化（PVC），那么它的数据（例如 Dashboard、Alert 配置等）是存储在 Pod 的临时文件系统里的，Pod 重启后数据就会丢失。现在如果你希望将来升级时不丢失这些配置，就需要先将 Grafana 的数据持久化到 PVC 上。

这里有几个注意点：

启用 PVC 后的数据问题：

现有数据不会自动迁移：如果你之前没有启用 PVC，那么当前 Grafana 的数据存储在容器内，启用 PVC 后默认会创建一个新的 PVC，Grafana 会使用新的数据目录，原有的临时数据不会自动被迁移过去。
数据丢失风险：因此，直接在配置中启用 PVC 后升级，如果没有对现有配置进行备份和迁移，原有 Dashboard 和 Alert 配置将无法保留。
正确步骤：

先备份现有数据：在进行任何变更之前，建议先手动导出 Dashboard 和 Alert 配置，或使用 Grafana 自带的备份工具进行备份。
手动迁移数据：你可以考虑将现有容器内的数据迁移到 PVC 中。方法可以是：
在 Pod 运行期间，将 /var/lib/grafana 目录内的数据复制出来，然后放到新建的 PVC 中。
或者先启用 PVC，同时保留原来的数据（例如，通过临时挂载 PVC 后，手动将数据拷贝进 PVC 中），确保 Grafana 使用的目录中包含现有配置。
配置 PVC 绑定：在 Helm Chart 的 values.yaml 中启用 Grafana 的持久化功能，最好指定 existingClaim 参数，这样可以指定一个你已经准备好数据的 PVC，确保升级后 Grafana 会使用该 PVC。例如：
yaml
コピーする
grafana:
  persistence:
    enabled: true
    existingClaim: your-grafana-pvc
如果没有 existingClaim 的话，Helm 会创建一个新的 PVC，根据配置中的存储大小和 StorageClass 来创建。
升级操作：

一旦你确保已有数据被正确备份或迁移到 PVC 中，再执行 helm upgrade 升级 Grafana 版本，此时新的 Grafana Pod 会挂载这个 PVC，从而继续使用上面的数据。
如果未做迁移而直接启用 PVC，那么升级后 Grafana 将从新的空 PVC 启动，原有的配置数据就无法恢复。
总结
先备份/导出数据：因为目前 Grafana 的数据是临时存储的，升级启用 PVC后不会自动迁移现有数据。
启用 PVC并迁移数据：确保新启用的 PVC 中包含现有的 Dashboard 和 Alert 数据，方法可以是手动迁移或利用 existingClaim 指定已有的数据卷。
再进行 Helm 升级：升级后的 Grafana 会挂载你指定的 PVC，从而保留之前的配置。
这样就可以在升级 Grafana 版本的同时，不丢失已有的 Alert 和 Dashboard 配置。
