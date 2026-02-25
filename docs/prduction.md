c
kubectl port-forward svc/argocd-server -n argocd 8080:443
kubectl port-forward svc/traefik -n kube-system 8081:80
kubectl port-forward svc/kibana-service 5601:80 -n tmcp-marketing



Unseal Key 1: xywHcnRYP7RlPUycBAHuJUqMqb5lqxE1uQ62UVgRAugb
Unseal Key 2: 74efgi9kneFHq+z/AsCVNdugSlyl9ouDH0dnPtjnnKtW
Unseal Key 3: jbBzA7Rp3tL59wbBpmFxxnONZ/AJCA5hDZ3D4uYao+RT
Unseal Key 4: uKNjvsGSx8cdidl02kft23Whw+A1TKi7jXoaJyylucX5
Unseal Key 5: fJfmrogcrvUfbSc2M1Sm0pnbn9knjWo8y39vQF+WYvHO

Initial Root Token: hvs.t7i73kNX1SedYVV25dVezbWz
sudo kubectl exec -it -n vault vault-0 -- vault login hvs.t7i73kNX1SedYVV25dVezbWz
-> Key                  Value
---                  -----
token                hvs.t7i73kNX1SedYVV25dVezbWz
token_accessor       qQS607DxcOlturP50mqywTkg
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
dangtung@dangs-Mac-mini ~ % 
# 1. Bật tính năng
sudo kubectl exec -it -n vault vault-0 -- vault auth enable kubernetes

# 2. Báo cấu hình K8s
sudo kubectl exec -it -n vault vault-0 -- sh -c 'vault write auth/kubernetes/config kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443"'

# 3. Tạo Policy
sudo kubectl exec -it -n vault vault-0 -- sh -c "echo 'path \"secret/data/tmcp/*\" { capabilities = [\"read\"] }' | vault policy write eso-policy -"

# 4. Tạo Role
sudo kubectl exec -it -n vault vault-0 -- vault write auth/kubernetes/role/eso bound_service_account_names=external-secrets bound_service_account_namespaces=external-secrets policies=eso-policy ttl=1h


kubectl exec -it -n vault vault-0 -- vault kv put secret/tmcp/agent POCKETBASE_PASSWORD="123qweasdzxc"
kubectl exec -it -n vault vault-0 -- vault kv put secret/tmcp/bridge POCKETBASE_PASSWORD="123qweasdzxc"

kubectl exec -it -n vault vault-0 -- vault kv put secret/tmcp/aiops-agent DISCORD_WEBHOOK_URL="https://discord.com/api/webhooks/1475801705152118865/m2HhYUyAvCl-UaKuWBaTKUpe-P62VtPNvuKkof002EfU8ZlPB86RarHVbAKgnfPvcJrh"

kubectl exec -it -n vault vault-0 -- \
   vault kv put secret/tmcp/kibana \
   ELASTICSEARCH_HOSTS='["http://elasticsearch-service:9200"]' \
   XPACK_ENCRYPTEDSAVEDOBJECTS_ENCRYPTIONKEY=$(openssl rand -hex 16) \
   XPACK_REPORTING_ENCRYPTIONKEY=$(openssl rand -hex 16) \
   XPACK_SECURITY_ENCRYPTIONKEY=$(openssl rand -hex 16)
