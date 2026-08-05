# Connect vault pod and run below command

```bash
vault operator init

# It will use to provide 5 keys and master key

vault operator unseal <sealed key1>
vault operator unseal <sealed key2>
vault operator unseal <sealed key3>
```