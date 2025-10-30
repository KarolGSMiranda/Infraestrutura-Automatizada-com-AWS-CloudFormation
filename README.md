# Implementando uma Infraestrutura Automatizada com AWS CloudFormation

Este repositório documenta minha jornada e aprendizado ao completar o desafio "Implementando uma Infraestrutura Automatizada com AWS CloudFormation". Este projeto faz parte da **Formação AWS Cloud Foundations** da DIO (Digital Innovation One).

O objetivo é evoluir do conceito de criar uma única "Stack" para a implementação de uma infraestrutura completa de forma automatizada, padronizada e repetível.


##  O "Porquê" da Automação: Infraestrutura como Código (IaC)

No desafio anterior, o foco era criar uma Stack. Agora, o foco é na **automação**. Isso significa eliminar a necessidade de intervenção manual e garantir que eu possa recriar minha infraestrutura exatamente da mesma forma, quantas vezes for necessário.

** Meu principal insight:** A virada de chave foi parar de tratar meus servidores como "animais de estimação" (cuidados um a um, com nomes e configurações manuais) e passar a tratá-los como "gado" (descartáveis, idênticos e gerenciados em grupo).

A automação com CloudFormation nos dá:
* **Repetibilidade:** Posso criar um ambiente de `desenvolvimento` hoje e um de `produção` amanhã, e ter 100% de certeza de que são idênticos.
* **Velocidade:** Provisionar uma arquitetura complexa (VPC, Subnets, Load Balancer, EC2s) leva minutos, em vez de horas de cliques no console.
* **Segurança e Conformidade:** Eu posso auditar meu template (`.yaml`) para garantir que todas as regras de segurança (como Security Groups) estão corretas, *antes* do deploy.
* **Versionamento:** Minha infraestrutura agora vive no Git. Eu sei *quem* mudou *o quê* e *quando*.

---

## 🛠️ Pilares da Automação no CloudFormation

Para alcançar a verdadeira automação, eu precisei ir além do básico e usar recursos mais avançados do CloudFormation.

### 1. Parâmetros (`Parameters`)
Esta é a principal ferramenta para tornar um template reutilizável.

* **O que é:** Permite que eu "pergunte" valores no momento do deploy (ex: `Qual o tipo da instância?`, `Qual o nome do ambiente (dev/prod)?`).
* **Meu uso:** Eu usei parâmetros para definir o `EnvironmentName` (ex: "dev") e o `InstanceType`. Assim, o mesmo template pode criar um ambiente de dev (com `t2.micro`) e um de prod (com `t3.large`).

### 2. Mapeamentos (`Mappings`)
São usados para lógicas condicionais simples, baseadas em chaves (como um dicionário ou mapa).

* **O que é:** Permite definir valores que mudam com base em uma chave, como a Região da AWS.
* **Meu uso:** Eu usei `Mappings` para selecionar o `AMI-ID` (ID da imagem da máquina) correto. O AMI do Amazon Linux é diferente em cada região. Com o Mappings, meu template automaticamente seleciona o AMI certo para a região onde o Stack está sendo criado (usando a pseudo-variável `AWS::Region`).

### 3. Saídas (`Outputs`) e Referências Cruzadas
Um ambiente automatizado raramente vive em um único Stack.

* **O que é:** A seção `Outputs` de um Stack permite "exportar" valores (como o ID de uma VPC ou o DNS de um Load Balancer).
* ** Insight:** Foi aqui que a automação brilhou. Eu quebrei minha infraestrutura em dois Stacks:
    1.  **Stack de Rede (Fundação):** Cria a VPC, Subnets, Internet Gateway. Ele usa `Outputs` para exportar o `VPC-ID` e os `Subnet-IDs`.
    2.  **Stack de Aplicação:** Cria o Load Balancer e as Instâncias EC2. Ele *importa* os valores do Stack de Rede usando a função `!ImportValue`.
* **Resultado:** Eu posso atualizar minha aplicação (Stack 2) sem *nunca* tocar na minha rede (Stack 1). Isso é desacoplamento!

### 4. Change Sets (Conjuntos de Alterações)
A automação é perigosa se for cega.

* **O que é:** Antes de *atualizar* um Stack, eu aprendi a **não** clicar em "Update" direto. Eu crio um **Change Set**.
* **Meu uso:** O Change Set é um *preview*. O CloudFormation analisa meu template modificado e me dá um relatório: `[Modificar] Recurso X`, `[Adicionar] Recurso Y`, `[Remover] Recurso Z`. Isso me dá a confiança de aprovar uma mudança em um ambiente automatizado sem medo de quebrar tudo.

---

##  Implementação Prática: Arquitetura Automatizada

Para este desafio, meu objetivo foi automatizar a implantação de uma arquitetura de aplicação web resiliente.

**Arquitetura-Alvo:**
* Uma VPC customizada.
* Subnets públicas e privadas em múltiplas Zonas de Disponibilidade.
* Um Application Load Balancer (ALB) na subnet pública.
* Um Auto Scaling Group (ASG) de instâncias EC2 rodando um servidor web simples (ex: Apache) nas subnets privadas.
* Security Groups granulares (ex: ALB só aceita porta 80, EC2s só aceitam tráfego do ALB).

**Como a automação foi aplicada:**
1.  O template mestre (`main.yaml`) foi criado.
2.  Usei `Parameters` para pedir o nome do ambiente e o par de chaves SSH.
3.  Usei `Mappings` para selecionar o AMI correto da região.
4.  Defini os `Resources`: VPC, Subnets, Rotas, ALB, Launch Template, ASG e Security Groups.
5.  Usei `Outputs` para exportar o DNS do Load Balancer, para que eu pudesse acessar o site.

---

##  Lições Aprendidas 

1.  **O `ROLLBACK` é uma feature, não um bug:** Meu Stack falhou algumas vezes. O CloudFormation automaticamente deletou *tudo* o que tinha criado (o `ROLLBACK`). No início, achei frustrante. Depois, percebi que isso é uma garantia de consistência. A automação prefere falhar e reverter a deixar um ambiente "meio-feito" e inconsistente.
2.  **A "Deriva" (`Drift`) é o inimigo:** O maior perigo da automação é alguém (eu mesmo, no futuro) fazer uma mudança manual no console (ex: "só vou abrir essa porta aqui rapidinho"). Isso causa "Drift" (desvio), onde o template não reflete mais a realidade. A automação exige disciplina: *toda* mudança deve ser feita *via template*.
3.  **`cfn-lint` economiza horas:** Comecei a usar a ferramenta `cfn-lint` (um validador local) no meu VS Code. Ele aponta erros de sintaxe e más práticas *antes* de eu tentar o deploy. Isso economizou horas de espera por falhas de `ROLLBACK`.

##  Próximos Passos (Além da Automação)

A infraestrutura está automatizada, mas o *processo* de deploy ainda foi manual. O próximo passo lógico é a **Entrega Contínua (CI/CD)**:

* Integrar este repositório com o **GitHub Actions** ou **AWS CodePipeline**.
* Criar um pipeline que, a cada `git push` na branch `main`:
    1.  Valide o template (com `cfn-lint`).
    2.  Crie um `Change Set` automaticamente.
    3.  (Opcional) Espere por uma aprovação manual.
    4.  Execute o `Change Set` para aplicar a mudança na infraestrutura.
