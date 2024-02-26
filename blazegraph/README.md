# Deploy Blazegraph

Complete example to deploy Blazegraph using Docker, and bulk load a large dataset

Replace `localhost` by your URL, mount the data to load in a volume on `/data` inside the container, and run:

```bash
curl -X POST --header 'Content-Type:text/plain' \
    --data-binary @dataloader.txt \
    https://localhost:8080/bigdata/dataloader
```

Blazegraph optim: https://github.com/polifonia-project/registry_app/blob/cf87aee056c35406771b0a19bd2b0f42dc849004/blaze_vm.properties
