# Slack & Prometheus AlertManager integration

After enabling Prometheus in Platform9 Managed Kubernetes, you will see that several services are created in the pf9-monitoring namespace
```bash
kubectl get svc -n pf9-monitoring
```
```bash
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
alertmanager-operated   ClusterIP   None            <none>        9093/TCP,9094/TCP,9094/UDP   3d5h
comms-proxy             ClusterIP   10.21.65.182    <none>        9900/TCP                     3d5h
grafana-ui              ClusterIP   10.21.174.118   <none>        80/TCP                       3d5h
kube-state-metrics      ClusterIP   None            <none>        8443/TCP,8081/TCP            56d
node-exporter           ClusterIP   None            <none>        9100/TCP                     56d
prometheus-operated     ClusterIP   None            <none>        9090/TCP                     3d5h
sys-alertmanager        ClusterIP   10.21.55.181    <none>        9093/TCP                     3d5h
sys-prometheus          ClusterIP   10.21.155.239   <none>        9090/TCP                     3d5h
sysalert-4pu0u5me       ClusterIP   10.21.174.71    <none>        9093/TCP                     3d5h
system-yc1jobvw         ClusterIP   10.21.39.200    <none>        9090/TCP    
```
You can easily connect to the Prometheus UI by leveraging the port-forwarding mechanism in kubectl. Once you issued the below command, you will be able to connect to the Prometheus UI from your local machine.

```bash
kubectl port-forward svc/sys-prometheus -n pf9-monitoring 9090:9090
```
You can review the out of the box alert rules by going to the Alerts page.  You will notice that there is one active Watchdog alert. This is an alert meant to ensure that the entire alerting pipeline is functional. This alert is always firing, therefore it should always be firing in AlertManager and always fire against a receiver. This is very convenient in case you want to test integrations with Slack, PagerDuty, email and others.

The steps we need to follow in order to integrate AlertManager with Slack are:
1. Create a Slack channel where you want to fire the Alerts;
2. Grab the Slack WebHook URL;
3. Create an AlertManager template;
4. Encode the AlertManager template;
5. Update the AlertManager-sysalert yaml.

## Create a Slack channel where you want to fire the Alerts
I created a dedicated Slack channel named: alerts-pmk-freedom

## Grab the Slack WebHook URL
In the Slack administration settings you can create an incoming WebHook. This will generate a WebHook URL which you will need in the AlertManager template. 

## Create an AlertManager template
Create a new file (name is not important) where we will configure the Slack AlertManager template. For example: *alertmanager.yaml*

```yaml
global:
slack_api_url: '<<your slack webhook URL>>'

route:
receiver: 'slack-notifications'
group_by: [alertname, datacenter, app]

receivers:

name: 'slack-notifications'
slack_configs:

channel: '#alerts-pmk-freedom'. << update with your Slack channel name
text: 'https://platform9.com/wiki/alerts{ .GroupLabels.app }}/{{ .GroupLabels.alertname }}' << update with whatever you want to point to
```

This is a very easy notification template. You can find here more examples: https://prometheus.io/docs/alerting/notification_examples/

## Encode the AlertManager template
Encode your AlertManager template:

```bash
cat alertmanager.yaml | base64
```

## Update the AlertManager-sysalert yaml
Grab the name of the AlertManager secret:
```bash
kubectl get secret -n pf9-monitoring

NAME                             TYPE                                  DATA   AGE
alertmanager-sysalert            Opaque                                1      3d5h
default-token-g8mr9              kubernetes.io/service-account-token   3      56d
grafana-datasources              Opaque                                1      3d5h
kube-state-metrics-token-6pbmj   kubernetes.io/service-account-token   3      56d
node-exporter-token-tjmkd        kubernetes.io/service-account-token   3      56d
prometheus-system                Opaque                                1      3d5h
system-prometheus-token-jtng2    kubernetes.io/service-account-token   3      56d
```
Put the encoded string into the alertmanager.yaml value of the alertmanager-sysalert secret:

```bash
kubectl -n pf9-monitoring edit secret alertmanager-sysalert -o yaml
```

## Review Alert Configuration and Slack notifications

