# Arquitetura do Sistema de Geração de Relatórios (XLSX/PDF)

## 📌 Visão Geral
Este sistema foi projetado para a geração assíncrona de relatórios nos formatos **XLSX e PDF**, garantindo **escalabilidade, resiliência e idempotência**. Ele utiliza **AmazonMQ** para processamento assíncrono, armazenamento distribuído para persistência dos arquivos e uma API REST para interação com os clientes.

---

## 🏛️ Arquitetura Proposta

### **Fluxo de Requisição**
1. O cliente envia uma requisição **`POST /generate`** para solicitar um novo relatório.
2. A requisição passa pelo **API Gateway**, que roteia para uma **AWS Lambda** do **Report Service (NestJS)**.
3. O serviço valida a solicitação, armazena um registro inicial no **PostgreSQL/MySQL** e publica uma mensagem na fila do **AmazonMQ**.
4. Uma **AWS Lambda Worker** consome a mensagem, gera o relatório (XLSX/PDF) e armazena no **S3 (ou similar)**.
5. O Worker atualiza o status do relatório no **banco de dados**.
6. O cliente pode consultar o status pelo endpoint **`GET /status/{id}`**, passando novamente pelo **API Gateway**.
7. Quando o relatório estiver pronto, um link para download é retornado.

### **Componentes Principais**
- **API Gateway**: Gerencia o tráfego das requisições HTTP e invoca as Lambdas corretas.
- **Report Service (Lambda NestJS)**: Processa as requisições e interage com AmazonMQ e o banco de dados.
- **AmazonMQ**: Gerencia a fila de processamento assíncrono.
- **Worker de Geração (Lambda)**: Responsável por processar os arquivos e armazená-los.
- **Banco de Dados (PostgreSQL/MySQL)**: Armazena o estado da geração dos relatórios.
- **Armazenamento (S3/MinIO)**: Guarda os arquivos gerados.

---

## 🛠️ Justificativa Técnica

### **Escolha do AmazonMQ ao invés de Amazon SQS**
| Critério              | AmazonMQ | Amazon SQS |
|----------------------|-----------|-----------|
| **Latência**         | Muito baixa (TCP) | Maior (HTTP) |
| **Controle**         | Maior flexibilidade para **routing e reprocessamento** | Simples, sem suporte avançado a routing |
| **Persistência**     | Controle total sobre expiração e reentrega | Fila pode reter mensagens até 14 dias |
| **Escalabilidade**   | Requer configuração manual | Escala automaticamente |
| **Complexidade**     | Requer deploy e tuning | Simples e gerenciado pela AWS |

Decidimos utilizar o **AmazonMQ** porque:
- O sistema exige **baixa latência** e **entrega rápida de mensagens**.
- Precisamos de um **controle granular** sobre o roteamento e processamento das mensagens.
- O AmazonMQ permite **priorizar tarefas** e gerenciar melhor o reprocessamento.
- Como os Workers rodam dentro de nossa infraestrutura, **não há necessidade de um sistema totalmente gerenciado** como o SQS.

Se a necessidade fosse **escalabilidade infinita sem gerenciamento**, o **SQS seria uma opção melhor**.

### **Escalabilidade e Concorrência**
- **Escalabilidade Horizontal**: O uso de **Lambdas** permite escalar de forma automática e sem necessidade de gerenciamento manual.
- **Fila de Mensagens**: AmazonMQ distribui as tarefas entre múltiplos Workers, evitando gargalos.
- **Armazenamento Distribuído**: O uso de **S3 ou MinIO** permite escalabilidade infinita para os arquivos.

### **Idempotência e Reprocessamento**
- **Garantia de Idempotência**: Cada requisição recebe um ID único, evitando a geração duplicada de relatórios.
- **Mensagens Persistentes**: AmazonMQ mantém as mensagens na fila até que um Worker as processe com sucesso.
- **Retries e Dead Letter Queue (DLQ)**: Em caso de falha, as mensagens são reenviadas ou movidas para uma fila de erros para análise posterior.

### **Cache e Performance**
- **Redis como Cache**: Utilizado para armazenar estados temporários e evitar consultas desnecessárias ao banco de dados.
- **CDN para Download**: Os arquivos podem ser entregues via **CloudFront**, garantindo baixa latência no acesso aos relatórios.

---

## 📊 Benefícios da Arquitetura
✅ **Alta Disponibilidade**: Distribuição de carga e filas garantem robustez.  
✅ **Baixa Latência**: AmazonMQ permite processamento rápido.  
✅ **Escalabilidade**: AWS Lambda escala automaticamente conforme a demanda.  
✅ **Resiliência**: O sistema se recupera de falhas automaticamente.  
✅ **Segurança**: O acesso aos arquivos é gerenciado via links assinados.  

---

## 🔗 Conclusão
Este design permite um **processamento eficiente, escalável e resiliente**, garantindo **baixo tempo de resposta e controle total** sobre a geração de relatórios. As decisões técnicas foram tomadas com foco em **desempenho, consistência e custo-benefício**.

Se houver necessidade de ajustes, a arquitetura pode ser refinada para suportar novas demandas. 🚀
