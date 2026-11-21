# ☁️ Relatório de Laboratório: Armazenamento no Microsoft Azure (Azure Storage)

Este documento reúne as anotações práticas, conceitos arquiteturais e boas práticas exploradas durante o laboratório de **Azure Storage** na plataforma **DIO**. O objetivo é consolidar o conhecimento sobre provisionamento, serviços de armazenamento, conectividade, migração e transferência de dados em nuvem.

---

## 1. Criação e Configuração da Conta de Armazenamento (Storage Account)

A **Conta de Armazenamento (Storage Account)** fornece um namespace exclusivo no Azure para seus dados, acessível de qualquer lugar do mundo via HTTP ou HTTPS.

### 1.1. Assinatura e Detalhes da Instância
* **Assinatura (Subscription) e Grupo de Recursos (Resource Group):** Define o limite de faturamento, governança e ciclo de vida dos recursos.
* **Nome da Conta:** Deve ser globalmente único (entre 3 e 24 caracteres, apenas letras minúsculas e números), pois compõe a URL pública do serviço (ex.: `https://<nomeconta>.blob.core.windows.net`).
* **Região:** Deve ser escolhida estrategicamente próxima aos usuários finais ou aos recursos computacionais (VMs, AKS, App Services) para minimizar a latência de rede e cumprir requisitos de conformidade (LGPD/GDPR).

### 1.2. Camadas de Desempenho: Standard vs. Premium
| Característica | Standard | Premium |
| :--- | :--- | :--- |
| **Mídia Física** | Discos magnéticos (HDD) | Unidades de estado sólido (SSD) de alta performance |
| **Latência** | Latência padrão (milissegundos) | Baixíssima latência (sub-milissegundo) |
| **Custos** | Baixo custo por GB armazenado | Custo mais elevado por GB, otimizado para IOPS |
| **Serviços Suportados** | Blobs, Files, Queues, Tables | Block Blobs, Page Blobs, File Shares (específicos por tipo de conta) |
| **Cenários Recomendados** | Backups, arquivamento, arquivos estáticos (imagens/vídeos), data lakes, filas e dados não críticos em tempo real. | Bancos de dados de alta transação, cargas de trabalho críticas de I/O intensivo, VMs de produção (Page Blobs) e compartilhamentos corporativos de alta velocidade. |

### 1.3. Opções de Redundância (Alta Disponibilidade e Durabilidade)
* **LRS (Locally Redundant Storage):** Replica os dados 3 vezes dentro de um único datacenter na região primária. Menor custo, protege contra falhas de hardware local.
* **ZRS (Zone-Redundant Storage):** Replica os dados de forma síncrona em 3 Zonas de Disponibilidade (datacenters fisicamente separados) na mesma região.
* **GRS (Geo-Redundant Storage):** Combina LRS na região primária com replicação assíncrona para uma região secundária pareada a centenas de quilômetros.
* **GZRS (Geo-Zone-Redundant Storage):** Combina ZRS na região primária com replicação geográfica para a região secundária (máxima disponibilidade e durabilidade).

---

## 2. Serviços Core do Azure Storage

### 2.1. Azure Blob Storage (Armazenamento de Objetos)
Projetado para armazenar grandes quantidades de dados não estruturados (textos, imagens, vídeos, backups, logs).
* **Conectividade:** Acesso via REST API, SDKs, Azure Portal, Storage Explorer e URLs diretas com controle de acesso anônimo ou autenticado (SAS Token / Microsoft Entra ID).
* **Camadas de Acesso (Access Tiers):** *Hot* (acesso frequente), *Cool* (acesso infrequente, mín. 30 dias), *Cold* (mín. 90 dias) e *Archive* (arquivamento offline de longo prazo, mín. 180 dias).
* **Recuperação e Proteção de Dados:**
  * **Soft Delete (Exclusão Temporária):** Mantém blobs e contêineres excluídos por um período de retenção configurável (ex.: 7 a 365 dias).
  * **Versionamento de Blob:** Preserva automaticamente versões anteriores sempre que um objeto é modificado ou sobrescrito.
  * **Point-in-Time Restore (PITR):** Permite restaurar conjuntos de blobs para um estado exato no passado.

### 2.2. Azure Files (Compartilhamento de Arquivos)
Compartilhamentos de arquivos totalmente gerenciados na nuvem, acessíveis simultaneamente por clientes locais e na nuvem.
* **Tecnologias e Protocolos:** Suporta os protocolos padrão da indústria **SMB (Server Message Block)** e **NFS (Network File System)**.
* **Backup:** Integração nativa com o **Azure Backup** (Recovery Services Vault), permitindo agendamento de snapshots e restauração granular de arquivos ou compartilhamentos inteiros.
* **Autenticação:** Suporte a chaves de acesso da conta, SAS Tokens e autenticação baseada em identidade via **Active Directory Domain Services (AD DS)** local, **Microsoft Entra Domain Services** ou **Microsoft Entra Kerberos**.
* **⚠️ Desafio de Conectividade (Porta TCP 445 - SMB):**
  * **O Problema:** O protocolo SMB utiliza a porta **TCP 445**. Muitos provedores de internet (ISPs) bloqueiam a porta 445 de saída por questões de segurança histórica contra ataques de rede.
  * **Soluções Recomendadas:**
    1. Configurar uma **VPN Point-to-Site (P2S)** ou **Site-to-Site (S2S)** para o Azure Virtual Network, encapsulando o tráfego SMB dentro do túnel seguro.
    2. Utilizar o **Azure File Sync** em um Windows Server local (que sincroniza via **HTTPS / porta 443**, geralmente liberada pelos provedores) e servir os arquivos localmente na rede interna via SMB.

### 2.3. Azure Queue Storage (Filas de Mensagens)
Serviço para armazenamento de grande volume de mensagens acessíveis de qualquer lugar via chamadas autenticadas HTTP/HTTPS.
* **Arquitetura:** Ideal para desacoplar componentes de aplicações distribuídas e processamento assíncrono (ex.: produtores inserem tarefas na fila e workers consomem sob demanda).
* **Estrutura de URL:**
  ```text
  https://<nome-da-conta>.queue.core.windows.net/<nome-da-fila>
  ```
* **Características:** Cada mensagem pode ter até **64 KB** e uma fila pode conter milhões de mensagens, limitadas apenas pela capacidade total da conta de armazenamento.

### 2.4. Azure Table Storage (Armazenamento NoSQL Chave-Valor)
Armazenamento de dados estruturados não relacionais (NoSQL) sem esquema rígido (*schemaless*).
* **Estrutura:** Organizado em tabelas contendo entidades (linhas), onde cada entidade possui um conjunto de propriedades (colunas), sendo obrigatórias a `PartitionKey` (define a partição lógica/física para escalabilidade) e a `RowKey` (identificador único dentro da partição).
* **Cenários:** Catálogos de produtos, perfis de usuários, metadados de dispositivos IoT e logs estruturados de baixo custo.

---

## 3. Migração de Dados para o Azure

### 3.1. O que pode ser migrado, Suportes e Cenários
O **Azure Migrate** funciona como um hub centralizado para descobrir, avaliar e migrar cargas de trabalho locais (*on-premises*) ou de outras nuvens para o Azure:
* **O que pode ser migrado:** Servidores físicos e virtuais (VMware, Hyper-V), bancos de dados (SQL Server, PostgreSQL, MySQL), aplicações web e grandes volumes de dados/arquivos.
* **Cenários Comuns:** Modernização de datacenters, encerramento de contratos de colocation, consolidação de servidores de arquivos e migração de ambientes legados sem interrupção crítica (replicação contínua antes do *cutover*).

### 3.2. Criação de um Projeto de Migração
1. No Portal do Azure, acessar o serviço **Azure Migrate** e criar um novo **Projeto de Migração** (vinculado a uma assinatura, grupo de recursos e região de metadados).
2. Adicionar as ferramentas de **Descoberta e Avaliação (Assessment)** para mapear dependências, dimensionar SKUs recomendadas no Azure e estimar custos.
3. Adicionar a ferramenta de **Migração (Migration and Modernization)**, implantar o *Appliance* no ambiente de origem, iniciar a replicação dos dados e executar o teste de migração antes da virada oficial.

### 3.3. Azure Data Box (Migração Offline)
Quando o volume de dados é massivo (dezenas de terabytes a petabytes) e a largura de banda da rede é insuficiente ou cara para transferência online:
* **Funcionamento:** A Microsoft envia um dispositivo físico robusto e criptografado (Data Box Disk, Data Box ou Data Box Heavy) para o datacenter do cliente.
* **Fluxo:** Os dados são copiados localmente em alta velocidade para o dispositivo, que é devolvido via transportadora para o datacenter do Azure, onde a equipe da Microsoft carrega os dados diretamente na Storage Account de destino (com criptografia AES de 256 bits e limpeza segura pós-carga).

---

## 4. Transferência e Gerenciamento de Dados: AzCopy vs. Storage Explorer

### 4.1. Cópia de Dados via Linha de Comando com `AzCopy`
O **AzCopy** é um utilitário de linha de comando de alto desempenho projetado para copiar dados de/para o Azure Blob e Azure Files.

* **Elementos Necessários:**
  * **Origem e Destino:** Caminhos locais (ex.: `C:\Dados\`) ou URLs remotas do Azure.
  * **Autenticação:** Feita via **Microsoft Entra ID** (`azcopy login`) ou anexando um **SAS Token (Shared Access Signature)** diretamente ao final da URL de destino.
  * **Flag `--recursive=true`:** Instrui o AzCopy a percorrer e copiar todos os subdiretórios e arquivos aninhados de forma recursiva.

* **Exemplo Prático de Comando:**
  ```bash
  azcopy copy "C:\LabDIO\Arquivos\*" "https://<minhaconta>.blob.core.windows.net/<meucontainer>?<SAS_TOKEN>" --recursive=true
  ```

### 4.2. Diferença entre AzCopy e Azure Storage Explorer
| Aspecto | AzCopy | Azure Storage Explorer |
| :--- | :--- | :--- |
| **Interface** | Linha de comando (CLI) | Interface gráfica (GUI) multiplataforma (Windows, macOS, Linux) |
| **Uso Principal** | Automação, scripts de CI/CD, rotinas agendadas e transferências massivas de alta velocidade. | Exploração visual, upload/download manual pontual, edição de metadados e geração rápida de SAS Tokens. |
| **Motor de Transferência** | Motor nativo de alta performance. | Utiliza o próprio **AzCopy** nos bastidores para executar as transferências de arquivos. |

### 4.3. Ambientes Suportados no Gerenciador de Armazenamento (Storage Explorer)
O **Azure Storage Explorer** permite conectar e gerenciar recursos em múltiplos ambientes:
* **Azure Global (Nuvem Pública):** Assinaturas comerciais padrão.
* **Nuvens Soberanas / Governamentais:** Azure China, Azure US Government.
* **Azure Stack / Ambientes Híbridos:** Conexão com instâncias locais do Azure Stack Hub.
* **Emuladores Locais (Desenvolvimento):** Integração direta com o **Azurite** (emulador local moderno do Azure Storage) para testes offline sem custos na nuvem.
```

Se quiser adicionar prints, comandos específicos que você executou no portal ou ajustar alguma seção para publicar no seu GitHub/DIO, é só avisar!
