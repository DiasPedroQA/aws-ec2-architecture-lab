# 🚀 Desafio DIO – Arquitetura de Instâncias EC2 na AWS

![AWS](https://img.shields.io/badge/AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)
![EC2](https://img.shields.io/badge/EC2-FF6600?style=for-the-badge&logo=amazonec2&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)
![Status](https://img.shields.io/badge/Status-Concluído-2E8B57?style=for-the-badge)

---

## 📋 O que eu precisava fazer?

A missão era clara: desenhar e documentar uma arquitetura na AWS que mostrasse que eu entendi os conceitos de gerenciamento de instâncias **EC2**, criação de **AMIs personalizadas** e uso de **Snapshots EBS**. Não bastava só jogar uns ícones no draw.io — eu precisava mostrar uma estrutura realista, que fizesse sentido para uma aplicação web de verdade.

---

## 🧠 Como eu pensei nessa arquitetura?

Quando comecei a rabiscar o desenho, imaginei o seguinte cenário: *"e se eu tivesse que hospedar uma aplicação web de uma startup que está começando agora, mas já pensando em crescer rápido?"*

A resposta veio naturalmente: eu precisava de uma estrutura que fosse **segura** (ninguém quer servidor web exposto diretamente à internet), **escalável** (porque tráfego pode explodir de uma hora pra outra) e com **backup garantido** (porque perder dados é pesadelo de qualquer dev).

Foi aí que montei essa arquitetura, componente por componente, pensando em cada decisão.

---

## 🖼️ Diagrama da Arquitetura

Aqui está o resultado final do desenho que criei no **draw.io** — com fundo escuro porque, convenhamos, fica bem mais estiloso e destaca melhor os componentes coloridos da AWS:

![Arquitetura AWS EC2](images/diagrama.png)

> 💡 *Dica: clique na imagem para expandir. O arquivo-fonte `.drawio` também está na pasta `images/` se quiser editar.*

---

## 🧱 Dissecando cada componente

### 1. VPC e Internet Gateway — o isolamento da rede

Eu comecei criando uma **VPC com CIDR 10.0.0.0/16**. Por quê? Porque isolar nossa infraestrutura em uma rede própria é o primeiro passo para ter controle sobre o que entra e sai. E claro, um **Internet Gateway** para permitir que o tráfego da internet chegue até o Load Balancer.

> *"Sem VPC bem definida, a casa cai antes mesmo de subir o primeiro servidor."*

---

### 2. Sub-redes e Zonas de Disponibilidade — a base da alta disponibilidade

Aqui eu dividi a VPC em duas **Availability Zones** (AZ1 e AZ2). Em cada AZ, coloquei uma sub-rede pública e uma privada. O racional foi simples: se uma AZ cair (sim, isso acontece), a outra segura o tranco.

| Zona de Disponibilidade | Sub-rede Pública         | Sub-rede Privada          |
|-------------------------|--------------------------|---------------------------|
| **AZ1**                 | 10.0.1.0/24 (ALB)        | 10.0.2.0/24 (EC2 + RDS)   |
| **AZ2**                 | 10.0.3.0/24 (Bastion)    | 10.0.4.0/24 (EC2 + RDS)   |

> *"Sub-rede pública pra receber o mundo, sub-rede privada pra proteger o que importa."*

---

### 3. Application Load Balancer (ALB) — o porteiro da aplicação

Coloquei um **ALB** na sub-rede pública da AZ1. Ele é o único ponto de contato com a internet nas portas **80/443**. O Security Group dele é bem restritivo: só aceita HTTP/HTTPS de qualquer lugar. Nada de SSH, nada de porta aberta indevida.

**Por que um ALB e não um Classic LB?** Porque o ALB trabalha na camada 7, entende HTTP, faz roteamento por path, host... bem mais flexível.

---

### 4. Bastion Host — a porta dos fundos (controlada)

Ninguém acessa servidor em sub-rede privada diretamente. Criei um **Bastion Host** na sub-rede pública da AZ2, com um Security Group que só aceita SSH **do meu IP doméstico**. Ou seja, se eu não estiver em casa, nem eu entro. Isso é segurança de verdade.

> *"Bastion é tipo aquela chave reserva escondida no jardim — só eu sei onde está."*

---

### 5. Auto Scaling Group (ASG) e Launch Template — crescimento inteligente

Aqui entra o coração da escalabilidade. Configurei um **Auto Scaling Group** que mantém de **1 a 3 instâncias EC2**, escalando quando a CPU passar de 70%. As instâncias são lançadas a partir de um **Launch Template** que referencia uma **AMI personalizada** chamada `MyWebApp`.

**O pulo do gato:** essa AMI já contém o Apache instalado, o código da aplicação deployado e as dependências resolvidas. Quando uma instância nova sobe, ela já nasce pronta pra produção — sem script de bootstrap gigante, sem demora.

---

### 6. Instâncias EC2 — os servidores web de fato

As instâncias ficam nas **sub-redes privadas** de ambas as AZs. Elas não têm IP público, não recebem tráfego direto da internet. Só aceitam conexões:

- Do **ALB** (tráfego web)
- Do **Bastion Host** (SSH administrativo)

Cada instância tem um **volume EBS gp3** anexado. Escolhi gp3 porque oferece bom desempenho com custo menor que io1/io2.

---

### 7. Snapshots EBS — o backup que me deixa dormir tranquilo

Configurei **snapshots automáticos** dos volumes EBS a cada 24 horas. Foi um dos tópicos mais importantes do curso: entender que EBS é redundante dentro da AZ, mas não é backup. Se alguém deletar um volume sem querer, só o snapshot salva.

No diagrama, representei isso com aquela linha tracejada vermelha ligando cada volume ao seu snapshot. É o "seguro de vida" dos dados.

---

### 8. Banco de Dados RDS Multi-AZ — redundância no dado

Escolhi um **RDS MySQL Multi-AZ**. Ele replica os dados entre as duas AZs automaticamente. Se a instância primária falhar, o failover acontece sem intervenção manual. O Security Group dele só aceita tráfego na porta **3306** vindo das instâncias EC2. Nada de acesso público ao banco — isso seria um desastre anunciado.

---

## 🔒 Segurança e Escalabilidade — o resumo do que importa

- **Camadas isoladas**: pública e privada. Cada uma com seu SG.
- **Security Groups como firewalls**: regras de entrada/saída bem definidas.
- **Auto Scaling**: aguenta picos de tráfego sem intervenção manual.
- **Snapshots + RDS Multi-AZ**: Disaster Recovery de verdade.
- **AMI personalizada**: deploy rápido, consistente e sem surpresas.

---

## 🎓 O que eu aprendi nesse processo

Esse desafio me fez consolidar vários conceitos que vi nas aulas, mas que só fazem sentido quando você desenha e explica com suas próprias palavras:

✅ Criar uma **AMI personalizada** não é só "tirar snapshot da instância" — é pensar no que deve estar pronto no boot.

✅ **VPC e sub-redes** são a espinha dorsal de qualquer arquitetura segura na nuvem.

✅ **Auto Scaling** sem Launch Template bem configurado é receita pra dor de cabeça.

✅ **Snapshots** são baratos e salvam vidas (de dados).

✅ Documentar a arquitetura é tão importante quanto construí-la — senão, daqui a 3 meses, nem eu vou lembrar o que cada coisa fazia.

---

## 🛠️ Como reproduzir isso (se quiser testar)

Se alguém quiser subir essa mesma infraestrutura na sua conta AWS, eu recomendo usar **CloudFormation** ou **Terraform** (quem sabe um próximo desafio?). Por enquanto, o diagrama e esse README já servem como mapa completo da arquitetura.

---

## 📂 Estrutura do Repositório

```bash
aws-ec2-architecture-lab/
├── README.md                 ← Você está aqui
└── images/
    ├── diagrama.drawio       ← Arquivo-fonte editável
    └── diagrama.png          ← Exportação em alta qualidade
```

---

## 🙋‍♂️ Considerações finais

Gostei muito do resultado final. O desenho ficou bonito, bem organizado e, principalmente, representa uma arquitetura que eu realmente implementaria em um ambiente de produção (com os devidos ajustes de custo, claro).

Se você chegou até aqui lendo tudo, parabéns! Significa que esse README cumpriu seu papel: documentar de forma clara, mas sem aquele monte de jargão que ninguém aguenta.

---

*Desafio proposto pela [DIO](https://www.dio.me/) como parte do módulo de Gerenciamento de Instâncias EC2 na AWS.* 🚀
