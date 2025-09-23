


Deploy the Open WebUI image.

gcloud run deploy openwebui \
--image=dyrnq/open-webui \ 
--cpu=8 --memory=32Gi \
--allow-unauthenticated \ 
--region $REGION \
--execution-environment=gen2 \
--set-env-vars=OLLAMA_BASE_URL=$OLLAMA_SERVICE_URL \
--set-env-vars=WEBUI_AUTH=false
