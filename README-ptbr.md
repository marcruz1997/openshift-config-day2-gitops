# Automatizando a criação de Applications no Argo CD com Helm

Quando começamos a trabalhar com **GitOps** e **Argo CD**, um dos primeiros desafios é organizar as aplicações que vamos gerenciar.  
Cada aplicação no Argo CD precisa de um recurso `Application`, que diz **de onde buscar os manifests**, **para onde aplicar** e **como sincronizar**.  

Criar esses `Applications` manualmente funciona, mas conforme o número de apps cresce, fica difícil manter tudo padronizado.  
É aqui que entra o **Helm**: podemos usar um **template genérico** para gerar vários `Applications` de forma automática.  

---

## Como funciona o template

O template percorre uma lista de aplicações definida no `values.yaml` e gera para cada uma delas um recurso `Application`:

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
