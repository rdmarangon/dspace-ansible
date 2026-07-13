# dspace-ansible

Automação Ansible para deploy do DSpace 9 em ambiente bare-metal com três servidores.

## Contexto: dois projetos, dois papéis

Este repositório e o `dspace-docker` trabalham em etapas distintas:

| Projeto          | Papel                                                                                                        | Quando usar                          |
| ---------------- | ------------------------------------------------------------------------------------------------------------ | ------------------------------------ |
| `dspace-docker`  | **Migração** — sobe o ambiente Docker, recebe o dump 6.3, migra o schema 6.3 → 7.6 → 9.x e gera o dump final | Uma vez, antes do go-live            |
| `dspace-ansible` | **Produção** — provisiona os três servidores bare-metal e recebe o dump já migrado                           | Deploy inicial e manutenção contínua |

O fluxo entre eles:

```
dspace-docker                              dspace-ansible
─────────────────────────────              ──────────────────────────────
 docker-compose-9.yml                       site.yml (provisiona servidores)
   └─ migra schema 6.3→9.x                      └─ restore_dump.yml
   └─ pg_dump -Fc dspace > dspace.dump  ──────►       └─ pg_restore + migrate ignored
                                                       └─ index-discovery -b
                                                       └─ oai import + sitemaps
```

O dump gerado pelo Docker é o único insumo que o Ansible precisa para colocar o repositório em produção.

---

## Arquitetura

```
┌─────────────────┐      ┌───────────────────────────┐      ┌──────────────────────┐
│   dspace-db     │      │     dspace-backend        │      │   dspace-frontend    │
│                 │      │                           │      │                      │
│  PostgreSQL 16+ │◄─────│  DSpace 9 (Tomcat 10)     │◄─────│  Angular SSR (Node)  │
│  porta 5432     │      │  porta 8080               │      │  porta 4000          │
│  acesso restrito│      │  + Solr 9.x               │      │  + nginx :80/:443    │
│  ao backend     │      │  porta 8983 (localhost)   │      │  proxy reverso       │
└─────────────────┘      └───────────────────────────┘      └──────────────────────┘
```

O nginx no servidor de frontend faz o SSL termination e roteia:

- `https://hostname/server` → backend:8080 (REST API)
- `https://hostname/sitemap*` → `/var/www/sitemaps/` (arquivos estáticos, ver [Sincronizar sitemaps](#sincronizar-sitemaps))
- `https://hostname/` → localhost:4000 (Angular SSR)

---

## Pré-requisitos

- Ansible 2.14+ na máquina de controle
- Ubuntu Server 24.04 LTS ou Debian 13 (Trixie) nos três servidores
- Acesso SSH com sudo nos três hosts
- Certificados SSL para o domínio (colocar em `roles/dspace_ui/files/ssl/`)
- Dump do banco de dados no formato custom do pg_dump (`pg_dump -Fc`)

> **PostgreSQL:** a variável `postgresql_version` em `inventory/group_vars/db.yml` deve corresponder à versão que a distro instala por padrão. Ubuntu 24.04 usa PG 16; Debian Trixie usa PG 18.
>
> **Java:** o playbook usa `openjdk-21-jdk-headless`, disponível tanto no Ubuntu 24.04 quanto no Debian Trixie.

Instale as coleções Ansible necessárias:

```bash
ansible-galaxy collection install -r requirements.yml
```

---

## Configuração inicial

### 1. Ajustar IPs

Edite `inventory/hosts.yml` com os IPs reais dos três servidores. Todos os demais arquivos derivam os IPs automaticamente a partir do inventário.

### 2. Ajustar variáveis

Os principais arquivos de variáveis:

| Arquivo                             | O que configurar                       |
| ----------------------------------- | -------------------------------------- |
| `inventory/group_vars/all.yml`      | `dspace_hostname`, `dspace_name`, SMTP |
| `inventory/group_vars/db.yml`       | `postgresql_version`                   |
| `inventory/group_vars/backend.yml`  | versões de Java/Maven/Solr/Tomcat      |
| `inventory/group_vars/frontend.yml` | `node_major_version`, `sitemaps_dir`   |

### 3. Criar o vault com a senha do banco

```bash
cp vault.yml.example vault.yml
# edite vault.yml e coloque a senha real em vault_db_password
ansible-vault encrypt vault.yml
```

### 4. Certificados SSL

Coloque os certificados em (usando o valor de `dspace_hostname` como nome de arquivo):

```
roles/dspace_ui/files/ssl/<dspace_hostname>.crt
roles/dspace_ui/files/ssl/<dspace_hostname>.key
```

---

## Deploy

### Completo (as três máquinas em sequência)

```bash
ansible-playbook site.yml --ask-vault-pass
```

### Por servidor individual

```bash
ansible-playbook playbooks/db.yml      --ask-vault-pass   # só banco
ansible-playbook playbooks/backend.yml --ask-vault-pass   # só backend + Solr
ansible-playbook playbooks/frontend.yml --ask-vault-pass  # só frontend
```

> **Atenção:** o build do Maven no backend leva ~20 minutos e o build do Angular no frontend leva ~15 minutos. A primeira execução é lenta; execuções subsequentes são idempotentes e rápidas.

### Ambiente de teste (máquina única)

`inventory/hosts-test.yml` roda os três servidores em localhost (conexão local, sem SSH).
O `dspace_hostname` está definido por host no inventário — ajuste para o IP da sua máquina
de teste antes de rodar (deve bater com o nome dos certificados SSL).

Coloque os certificados autoassinados correspondentes em:

```
roles/dspace_ui/files/ssl/<IP>.crt
roles/dspace_ui/files/ssl/<IP>.key
```

```bash
ansible-playbook site.yml -i inventory/hosts-test.yml --ask-vault-pass
```

---

## Restaurar dump

Este é o fluxo principal para colocar dados reais no DSpace 9.
O playbook cuida de toda a sequência automaticamente.

> **Migração completa requer dump + assetstore.** O banco contém apenas os metadados;
> os arquivos físicos dos itens (PDFs, imagens etc.) ficam em `/dspace/assetstore/` no
> servidor backend. Sem o assetstore os itens aparecem no sistema mas os bitstreams não
> são acessíveis. Sincronize o assetstore **antes** de subir o Tomcat (ver
> [Sincronizar assetstore](#sincronizar-assetstore)).

Gere o dump no servidor de origem com:

```bash
pg_dump -Fc -U dspace dspace > dspace.dump
```

Depois rode o playbook passando o caminho local do arquivo:

```bash
ansible-playbook playbooks/restore_dump.yml \
  --extra-vars "dump_local_path=/home/rodrigo/dspace.dump" \
  --ask-vault-pass
```

O dump deve estar no formato final desejado antes de rodar o playbook.

O que o playbook faz, na ordem:

1. Para o Tomcat
2. Dropa o banco existente e recria vazio
3. Importa o dump com `pg_restore`
4. Normaliza as URIs de handle: reescreve hosts internos/de teste (`dump_rewrite_internal_hosts` em `all.yml`) para `dspace_ui_url`, preservando referências a repositórios externos
5. Roda `dspace database migrate ignored`
6. Roda `index-discovery -b` (pode levar horas em acervos grandes)
7. Roda `oai import`
8. Gera sitemaps
9. Sobe o Tomcat

Após concluir, sincronize os sitemaps para o frontend:

```bash
ansible-playbook playbooks/sync_sitemaps.yml --ask-vault-pass
```

### Instalação limpa (sem dump)

O playbook inicializa o schema do banco automaticamente na primeira instalação.
Um passo manual é necessário para criar o usuário administrador:

```bash
cd /tmp && sudo -u dspace /dspace/bin/dspace create-administrator
```

> O `cd /tmp` é necessário porque o usuário `dspace` não tem permissão de escrita no
> diretório corrente quando chamado via sudo.

---

## Verificação pós-deploy

```bash
# API retorna HTTPS e nome correto
curl -sk https://<hostname>/server/api | python3 -m json.tool | grep -E "dspaceUI|dspaceServer|dspaceName"

# Frontend responde (deve ser 200)
curl -sk -o /dev/null -w "%{http_code}" https://<hostname>/

# Solr: todos os cores saudáveis (deve retornar 200 quatro vezes)
for core in search statistics authority oai; do
  echo -n "$core: "
  curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8983/solr/$core/admin/ping
done
```

---

## Operações comuns

### Reindexar o Solr do zero

```bash
ansible-playbook playbooks/reindex.yml --ask-vault-pass
```

### Reimportar OAI-PMH

```bash
# substitua hosts.yml por hosts-test.yml para ambiente de teste
ansible -i inventory/hosts.yml backend -m command \
  -a "/dspace/bin/dspace oai import" \
  --become --become-user=dspace
```

### Gerar sitemaps manualmente

```bash
# substitua hosts.yml por hosts-test.yml para ambiente de teste
ansible -i inventory/hosts.yml backend -m command \
  -a "/dspace/bin/dspace generate-sitemaps" \
  --become --become-user=dspace
```

Após gerar, rode `ansible-playbook playbooks/sync_sitemaps.yml --ask-vault-pass` para enviar ao frontend.

### Sincronizar assetstore

```bash
ansible-playbook playbooks/sync_assetstore.yml \
  --extra-vars "assetstore_src=/home/rodrigo/assetstore" \
  --ask-vault-pass
```

O playbook usa rsync incremental — a primeira execução é lenta (pode ser dezenas de GB), as seguintes transferem apenas o que mudou.

### Sincronizar sitemaps

```bash
ansible-playbook playbooks/sync_sitemaps.yml --ask-vault-pass
```

Rode após cada `generate-sitemaps`. O playbook faz pull do backend para a máquina de controle e depois push para o frontend, sem exigir acesso direto entre os servidores.

### Verificar versão e status do banco

```bash
# substitua hosts.yml por hosts-test.yml para ambiente de teste
ansible -i inventory/hosts.yml backend -m command \
  -a "/dspace/bin/dspace version" --become --become-user=dspace

ansible -i inventory/hosts.yml backend -m command \
  -a "/dspace/bin/dspace database info" --become --become-user=dspace
```

---

## Notas técnicas

### Solr: criação dos cores

O `ant fresh_install` cria os diretórios de configuração em `/dspace/solr/` mas não registra os cores no Solr. O playbook copia os configs para `server/solr/configsets/` e cria cada core com `solr create_core -d`. Os dados do índice em si são sempre gerados do zero via `index-discovery -b` após o restore, então não há nada a preservar.

### pgcrypto

DSpace 9 exige a extensão `pgcrypto` no banco. O role instala `postgresql-contrib` (que a fornece) e habilita a extensão com `CREATE EXTENSION pgcrypto` no banco `dspace`.

### Ambiente de teste: não publicar nada externamente

O `testedspace.ufjf.br` é só rede interna UFJF — Google/harvesters não o alcançam, então
OAI, sitemaps e meta-tags do Google Scholar ficam inertes. O risco real de "publicar pra
fora" é o **registro automático de DOI** (DataCite/Crossref): depositar um item com o
minting ligado cria um DOI **real** na agência, independente de rede. Por isso o
`local.cfg` mantém:

```properties
identifier.doi.enable = false
```

> **Ao virar produção:** para emitir DOIs de verdade, ligue `identifier.doi.enable = true`
> e configure as credenciais do provedor (DataCite/Crossref) em `local.cfg`. Enquanto for
> teste/homologação, **deixe `false`** para não registrar DOIs reais por engano.

### Configuração de submissão (formulários de depósito)

Os campos do formulário de depósito, dropdowns e o mapeamento coleção→formulário ficam em
`config/submission-forms.xml` e `config/item-submission.xml`. **Esses arquivos não vêm no
dump do banco** (o dump traz só os valores e o registro de metadados); o role os deploya a
partir de `roles/dspace/files/`. O perfil UFJF (tipos de documento, orientador, banca,
Lattes, CNPq etc.) vive aí. Alterações exigem `restart tomcat`.

### GeoIP para estatísticas

Controlado pela variável `geoip_enabled` em `inventory/group_vars/backend.yml` (padrão: `true`). Quando ativado, o Ansible baixa a base DB-IP do mês atual, configura o `local.cfg` automaticamente e cria um cron mensal no servidor backend para manter a base atualizada.

### filter-media (miniaturas e busca full-text)

Controlado por `filter_media_enabled` em `inventory/group_vars/backend.yml` (padrão: `true`). Instala ImageMagick + Ghostscript e cria um cron noturno rodando `dspace filter-media`, que gera as miniaturas (bundle `THUMBNAIL`) e extrai o texto dos PDFs (bundle `TEXT`) — este último é o que alimenta a **busca por conteúdo** no Solr. Não roda no depósito: itens novos ficam sem miniatura até o cron passar. A primeira execução processa todo o acervo (demorada); as seguintes só pegam o que é novo. Para rodar sob demanda:

```bash
sudo -u dspace /dspace/bin/dspace filter-media
```

---

## Estrutura do projeto

```
ansible/dspace/
├── ansible.cfg               ← remote_user, roles_path
├── requirements.yml          ← coleções Ansible
├── site.yml                  ← entry point: db → backend → frontend
├── vault.yml.example         ← template para criar vault criptografado
├── inventory/
│   ├── hosts.yml             ← IPs dos três servidores (produção)
│   ├── hosts-test.yml        ← IPs para ambiente de teste
│   └── group_vars/
│       ├── all.yml           ← vars comuns (hostname, timezone, mail)
│       ├── db.yml            ← postgresql
│       ├── backend.yml       ← java, maven, tomcat, solr
│       └── frontend.yml      ← node, dspace_ui_port, sitemaps_dir
├── playbooks/
│   ├── db.yml
│   ├── backend.yml
│   ├── frontend.yml
│   ├── restore_dump.yml      ← fluxo completo: limpa → importa → migra → reindex
│   ├── reindex.yml           ← só reconstrói o Solr
│   ├── sync_assetstore.yml   ← rsync incremental do assetstore
│   └── sync_sitemaps.yml     ← backend → controle → frontend
└── roles/
    ├── common/               ← timezone, locale pt_BR, UFW base, pacotes base
    ├── postgresql/           ← PG oficial, user/db, pgcrypto, pg_hba, UFW
    ├── solr/                 ← Solr 9.x, localhost-only, limits
    ├── dspace/
    │   ├── tasks/
    │   │   ├── prerequisites.yml  ← Java 21, Maven, Ant, user dspace
    │   │   ├── install.yml        ← download fonte, mvn package, ant fresh_install
    │   │   ├── tomcat.yml         ← tomcat10, JAVA_OPTS, ReadWritePaths, symlink
    │   │   ├── solr_cores.yml     ← copia configsets + cria cores
    │   │   ├── configure.yml      ← deploy local.cfg + forms de submissão, tomcat no grupo dspace
    │   │   └── geoip.yml          ← DB-IP download, symlink, cron (geoip_enabled)
    │   ├── files/
    │   │   ├── submission-forms.xml  ← perfil de submissão UFJF (campos, dropdowns, type-bind, vocabulário CNPq)
    │   │   ├── item-submission.xml   ← passos do depósito (inclui embargo/access e Creative Commons)
    │   │   └── default.license       ← licença de depósito (PT-BR)
    │   └── templates/
    │       └── local.cfg.j2       ← configuração principal do DSpace (db, solr, mail, urls)
    └── dspace_ui/
        ├── tasks/main.yml         ← Node 20, npm build (SSR), systemd, nginx
        └── templates/
            ├── config.prod.yml.j2    ← config do Angular SSR (ui.host, rest.host, ssl)
            ├── dspace-ui.service.j2  ← unit systemd do Node.js (--stack-size=65536)
            ├── nginx-global.conf.j2  ← http-level: gzip, limit_req_zone
            └── nginx-dspace.conf.j2  ← vhost: SSL, proxy, segurança, sitemaps
```

---

## Referências

- [DSpace 9.x Installation Guide](https://wiki.lyrasis.org/display/DSDOC9x/Installing+DSpace)
- [Apache Solr Installation](https://solr.apache.org/guide/solr/latest/deployment-guide/installing-solr.html)
