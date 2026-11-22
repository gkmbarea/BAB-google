# 🛡️ Proposta de Trabalho: Entendendo e Demonstrando o Binary Authorization for Borg (BAB)

> **Resumo Executivo**: Apresentação acadêmica e técnica sobre o modelo de segurança da infraestrutura da Google focado no **Binary Authorization for Borg (BAB)**, demonstrando como garantir que apenas binários autorizados e verificados sejam executados em clusters de produção.

---

## 🎯 1. Proposta e Objetivos do Trabalho

- **Contexto**: A segurança da cadeia de suprimentos de software (*Software Supply Chain Security*) abrange o caminho entre uma alteração no código e a versão que chega à produção. Uma compilação fora do ambiente confiável, um artefato trocado após a compilação ou uma configuração alterada sem revisão podem introduzir código ou comportamentos não autorizados.
- **Objetivo Geral**: Explicar a arquitetura do BAB e demonstrar como uma decisão de implantação pode ser vinculada a evidências sobre a origem do software e as etapas pelas quais ele passou.
- **Escopo Técnico**: Procedência de código e configuração, revisão por pares, compilação verificável, atestações assinadas, política por serviço, admissão, auditoria e tratamento de exceções.

---

## 🔐 2. O que é o BAB (Binary Authorization for Borg)?

- **Conceito Fundamental**: O BAB é um controle interno do Google integrado ao Borg, seu gerenciador de clusters. O Borg Prime consulta o BAB na admissão de um trabalho antes de conceder o certificado ALTS que representa sua identidade de serviço. Com a validação na implantação ativa, um trabalho fora da política aplicável é bloqueado, salvo pelo procedimento de emergência documentado.
- **Os Pilares Principais**:

  1. **Procedência e Rastreabilidade (*Code Provenance*)**: A procedência registra de qual revisão do código veio o artefato e qual sistema o gerou. Um resumo criptográfico (*digest*) identifica o conteúdo do artefato. Arquivos com o mesmo nome podem ter conteúdos e resumos diferentes.
  2. **Atestações Criptográficas (*Attestations*)**: Uma atestação declara um fato verificável sobre um artefato, como sua origem em uma compilação. A assinatura permite verificar quem emitiu a declaração e se ela foi alterada. A política didática deste trabalho também exige revisão por pares e testes automatizados, que podem incluir análise estática.
  3. **Verificação Estrita na Admissão (*Policy Enforcement*)**: O BAB avalia o código e a configuração conforme as regras da identidade do serviço. A própria política passa por revisão quando é criada ou modificada. A verificação contínua também avalia trabalhos já implantados e informa os responsáveis quando identifica divergências.
  4. **Modo de Emergência (*Break-Glass*)**: Como último recurso durante um incidente, o Google prevê uma exceção à validação na implantação com justificativa obrigatória. O BAB registra o trabalho e uma referência ao código implantado e notifica a equipe central de segurança e a equipe responsável pelo serviço.

> [!NOTE]
> A documentação pública descreve políticas que variam por serviço e controles compensatórios em alguns casos. A obrigatoriedade de testes, a seleção dos emissores e os campos do exemplo abaixo são escolhas da simulação.

---

## 🔄 3. Fluxo de Funcionamento e Demonstração Prática

### Fluxo da Cadeia de Suprimentos com BAB

```text
[Commit] → [Revisão por pares] → [Compilação e testes] → [Atestações assinadas]
                                                                 ↓
                                                      [Verificação na admissão]
                                                        ├─ Política atendida → [Deploy]
                                                        └─ Política violada → [Bloqueio]
                                                                                  ↓
                                                        [Incidente crítico? Break-glass]
                                                                                  ↓
                                                        [Exceção registrada e auditada]
```

### Política de exemplo

O serviço fictício **notificações** exige revisão do commit e da configuração, compilação oficial e aprovação nos testes. A política usa YAML ilustrativo para declarar as evidências esperadas.

```yaml
service: notificacoes
environment: production
enforcement: deny_on_policy_failure
required_evidence:
  - type: peer_review
    trusted_issuer: source-control
    binds_to: [source_commit, config_revision]
  - type: build_provenance
    trusted_issuer: official-builder
    binds_to: [source_commit, artifact_digest]
  - type: tests_passed
    trusted_issuer: ci-tests
    binds_to: [artifact_digest]
emergency:
  authorized_role: incident-responder
  require_incident_id: true
  require_justification: true
  notify: [security-team, service-owners]
  require_post_incident_review: true
```

### Guia de simulação

A atividade pode ser feita em sala, seguindo a política acima e usando a tabela para conferir os resultados. Trate `d1` e `d2` como identificadores fictícios de conteúdo e `c1` e `r1` como revisões fictícias de código e configuração.

1. **Prepare o pedido de implantação.** A equipe corrige um erro no serviço de notificações. O pedido indica o commit `c1`, a configuração `r1` e o artefato com digest `d1`.
2. **Associe as três evidências ao pedido.** O controle de versão confirma a revisão de `c1` e `r1`, o compilador oficial vincula `c1` a `d1` e o sistema de CI confirma os testes de `d1`. Para o exercício, considere as três assinaturas válidas.
3. **Faça o papel do verificador.** Confira a assinatura e se a identidade do emissor é confiável para aquele tipo de evidência. Compare os valores atestados com os valores do pedido. Autorize somente quando todas as verificações obrigatórias passarem.
4. **Mude um dado por vez.** Repita o cenário base com as alterações da tabela e registre a decisão e seu motivo.

| Cenário | Resultado esperado | Motivo |
| --- | --- | --- |
| Pedido original com todas as evidências válidas | ✅ Deploy autorizado | Revisões e digest correspondem aos valores atestados. |
| Falta a evidência de revisão | 🚫 Bloqueio | O commit e a configuração precisam de revisão comprovada. |
| Artefato com o mesmo nome, mas digest `d2` | 🚫 Bloqueio | As evidências de origem e testes referem-se a `d1`. |
| Assinatura válida de um compilador fora da lista de confiança | 🚫 Bloqueio | O emissor precisa ser autorizado para atestar a compilação. |
| Configuração alterada após a revisão | 🚫 Bloqueio | A configuração solicitada difere de `r1`. |
| Incidente crítico, evidência ausente e pedido de emergência autorizado e justificado | ⚠️ Deploy em caráter excepcional | O registro conserva o motivo da exceção e identifica o trabalho implantado. |
| Pedido de emergência sem justificativa | 🚫 Bloqueio | A política do exercício exige justificativa. |

> [!IMPORTANT]
> Na simulação de *break-glass*, registre o responsável, o identificador do incidente, o digest, a justificativa e as equipes notificadas. Após o incidente, revise a decisão e substitua a versão emergencial por uma versão que satisfaça a política, quando necessário.

---

## 📁 4. Estrutura do Repositório e Execução

O material da atividade está reunido no README. A simulação é manual e pode ser reproduzida com o diagrama, a política e os cenários da seção 3, sem instalar ferramentas ou acessar um cluster.

```text
.
└── README.md  # Proposta, política de exemplo e roteiro de simulação
```

---

## 📚 5. Referências Técnicas

- Google Research. [Large-scale cluster management at Google with Borg](https://research.google/pubs/large-scale-cluster-management-at-google-with-borg/)
- Google Cloud. [Binary Authorization for Borg (em português)](https://docs.cloud.google.com/docs/security/binary-authorization-for-borg?hl=pt-br)
- SLSA. [Especificação v1.2](https://slsa.dev/spec/v1.2/)
- in-toto. [Especificação do framework de atestações](https://github.com/in-toto/attestation/tree/main/spec)
- Google Cloud. [Visão geral do Binary Authorization para serviços do Google Cloud](https://cloud.google.com/binary-authorization/docs/overview)
