# Automatizando Applications do Argo CD com Helm

Se você já trabalhou com **Argo CD**, sabe que cada aplicação precisa de um recurso `Application` que informa:

- de onde buscar os manifests,
- para qual cluster e namespace aplicar,
- como sincronizar as alterações.

Quando a quantidade de apps começa a crescer, criar esses arquivos manualmente vira um pesadelo. É aí que o **Helm** entra em cena: com um template, você consegue gerar todos os `Applications` automaticamente a partir de um `values.yaml`.

---

## Como funciona o template

O template percorre a lista de aplicações definida no `values.yaml` e gera para cada uma delas um `Application`. Ele também trata labels, annotations, syncPolicy e finalizers, caso você queira customizar.  

```yaml
{{- $root := . -}}
{{- range $i, $app := .Values.applications }}
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: {{ $app.name | quote }}
  namespace: {{ $root.Values.argocd.namespace | quote }}
  {{- with $app.labels }}
  labels:
{{ toYaml . | nindent 4 }}
  {{- end }}
  {{- $finalizers := (default (list "resources-finalizer.argocd.argoproj.io") $app.finalizers) }}
  {{- if $finalizers }}
  finalizers:
  {{- range $finalizers }}
    - {{ . }}
  {{- end }}
  {{- end }}
  {{- with $app.annotations }}
  annotations:
{{ toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if $app.spec }}
{{ toYaml $app.spec | nindent 2 }}
  {{- else }}
  project: {{ default "default" $app.project | quote }}
  destination:
    server: {{ $app.destinationServer | quote }}
    {{- if $app.destinationNamespace }}
    namespace: {{ $app.destinationNamespace | quote }}
    {{- end }}
  {{- if $app.sources }}
  sources:
{{ toYaml $app.sources | nindent 4 }}
  {{- else }}
  {{- if $app.source }}
  source:
{{ toYaml $app.source | nindent 4 }}
  {{- else }}
  source:
    repoURL: {{ $app.repoURL | quote }}
    path: {{ $app.path | quote }}
    targetRevision: {{ default "HEAD" $app.targetRevision | quote }}
  {{- end }}
  {{- end }}
  {{- with $app.syncPolicy }}
  syncPolicy:
{{ toYaml . | nindent 4 }}
  {{- end }}
  {{- with $app.ignoreDifferences }}
  ignoreDifferences:
{{ toYaml . | nindent 4 }}
  {{- end }}
  {{- with $app.info }}
  info:
{{ toYaml . | nindent 4 }}
  {{- end }}
  {{- with $app.revisionHistoryLimit }}
  revisionHistoryLimit: {{ . }}
  {{- end }}
  {{- end }}
---
{{- end }}
