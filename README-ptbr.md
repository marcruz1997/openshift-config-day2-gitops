# Guia completo de Argo CD com Helm: AppProjects, Applications e Chart.yaml

Se você está começando com **Argo CD** e **GitOps**, provavelmente já percebeu que gerenciar várias aplicações e projetos manualmente pode ser complicado.
Neste guia, vamos mostrar como usar **Helm** para criar **AppProjects** e **Applications** de forma automática, padronizada e escalável, e como o `Chart.yaml` organiza seu chart.

---

## 1. Helm Chart.yaml: o coração do chart

O `Chart.yaml` é o arquivo principal de qualquer chart Helm. Ele define **informações básicas sobre o chart** e a aplicação que ele deploya.

Exemplo de `Chart.yaml`:

```yaml
apiVersion: v2
name: applications
description: Applications

type: application
version: 0.1.0
appVersion: "1.0"
```

* **apiVersion: v2** → Define que é um chart Helm versão 2.
* **name** → Nome do chart, usado para referenciar o pacote.
* **description** → Breve descrição do chart.
* **type** → Pode ser:

  * `application` → chart que gera templates para deploy de apps.
  * `library` → chart que fornece funções/utilitários para outros charts (não gera templates de deploy).
* **version** → Versão do chart, deve seguir [Semantic Versioning](https://semver.org/). Incrementa sempre que você muda templates ou configurações do chart.
* **appVersion** → Versão da aplicação que está sendo deployada pelo chart (não precisa seguir Semantic Versioning).

> Esse arquivo serve como referência central do Helm e ajuda a versionar tanto o chart quanto a aplicação que ele deploya.

---

## 2. AppProjects: organizando suas aplicações

Um **AppProject** funciona como um “container de aplicações” dentro do Argo CD. Ele permite:

* Agrupar várias aplicações (`Applications`) sob um mesmo projeto.
* Controlar **repositórios**, **clusters** e **namespaces** permitidos.
* Definir **permissões** através de roles e policies.
* Restringir recursos que podem ser manipulados.

### Diagrama do fluxo de AppProject

```mermaid
graph LR
A[AppProject] --> B[Applications]
A --> C[Roles e Policies]
A --> D[Repositórios permitidos]
A --> E[Clusters e Namespaces permitidos]
```

### Exemplo de template Helm para AppProject

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: {{ .Values.project.name }}
  namespace: {{ .Values.argocd.namespace }}
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: "{{ .Values.project.description | default "Projects apps cluster" }}"
  sourceRepos:
    {{- range .Values.project.sourceRepos }}
    - "{{ . }}"
    {{- end }}
  destinations:
    {{- range .Values.project.destinations }}
    - namespace: "{{ .namespace }}"
      server: "{{ .server }}"
    {{- end }}
  clusterResourceWhitelist:
    {{- range .Values.project.clusterResourceWhitelist }}
    - group: "{{ .group }}"
      kind: "{{ .kind }}"
    {{- end }}
  namespaceResourceBlacklist:
    {{- range .Values.project.namespaceResourceBlacklist }}
    - group: "{{ .group }}"
      kind: "{{ .kind }}"
    {{- end }}
  namespaceResourceWhitelist:
    {{- range .Values.project.namespaceResourceWhitelist }}
    - group: "{{ .group }}"
      kind: "{{ .kind }}"
    {{- end }}
  roles:
    {{- range .Values.project.roles }}
    - name: "{{ .name }}"
      description: "{{ .description | default "No description provided" }}"
      policies:
        {{- range .policies }}
        - "{{ . }}"
        {{- end }}
      groups:
        {{- range .groups }}
        - "{{ . }}"
        {{- end }}
      jwtTokens:
        {{- range .jwtTokens }}
        - iat: {{ .iat }}
        {{- end }}
    {{- end }}
```

### Exemplo de `values.yaml` para AppProject

```yaml
argocd:
  namespace: argocd

project:
  name: meu-projeto
  description: Projeto de apps de teste
  sourceRepos:
    - https://github.com/meu-org/*
  destinations:
    - namespace: app-namespace
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: ""
      kind: ConfigMap
  namespaceResourceBlacklist:
    - group: ""
      kind: Secret
  namespaceResourceWhitelist:
    - group: ""
      kind: Deployment
  roles:
    - name: devs
      description: Acesso para equipe de desenvolvimento
      policies:
        - "p, proj:meu-projeto:devs, applications, *, meu-projeto/*, allow"
      groups:
        - developers
```

---

## 3. Applications: declarando suas aplicações

Um **Application** no Argo CD define:

* De qual repositório Git buscar os manifests.
* Em qual cluster e namespace aplicar.
* Como sincronizar as alterações.

### Diagrama do fluxo de Applications

```mermaid
graph LR
A[Application] --> B[Repositório Git]
A --> C[Cluster de destino]
A --> D[Namespace de destino]
A --> E[SyncPolicy]
```

### Template Helm para Applications

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
```

### Exemplo de `values.yaml` para Applications

```yaml
argocd:
  namespace: argocd

applications:
  - name: minha-app
    project: meu-projeto
    destinationServer: https://kubernetes.default.svc
    destinationNamespace: app-namespace
    repoURL: https://github.com/meu-org/minha-app.git
    path: deploy
    targetRevision: main
    syncPolicy:
      automated: {}
```

---

## 4. Benefícios de usar Helm com Argo CD

* **Escalabilidade**: adiciona novos projetos e aplicações sem duplicar YAML.
* **Padronização**: todos os recursos seguem a mesma estrutura.
* **Segurança**: controla repositórios, clusters, namespaces e recursos permitidos.
* **Controle de acesso**: roles e policies definem permissões de forma centralizada.
* **Facilidade de manutenção**: basta atualizar o `values.yaml` para refletir mudanças.
* **Versionamento**: o `Chart.yaml` ajuda a versionar o chart e a aplicação separadamente.

---

## 5. Conclusão

Com esse conjunto de templates e o `Chart.yaml`:

1. Você organiza **AppProjects** para agrupar aplicações e definir permissões.
2. Cria **Applications** de forma padronizada, declarativa e escalável.
3. Versiona seu chart e sua aplicação com `Chart.yaml`.
4. Mantém segurança, controle de acesso e facilidade de manutenção.

A combinação **Helm + Argo CD** é ideal para equipes que usam GitOps em Kubernetes, permitindo **automatização, padronização e escalabilidade** de maneira confiável.
