# Arquitetura

[← Voltar ao README](../README.md)

## Visão geral

```
Clientes (PWA · Android TWA · Electron)
        │
        ▼
Next.js 15 na Vercel ── middleware (sessão, papel, assinatura, senha pendente)
        │
        ├── Supabase: Postgres + RLS · Auth · Storage
        ├── n8n: workflows de IA (Whisper, GPT-4o)
        ├── Stripe: checkout, portal, webhooks
        └── Resend: e-mails transacionais
```

Uma única base Next.js atende as três plataformas. O app Android é uma *Trusted Web Activity* validada
por Digital Asset Links (`/.well-known/assetlinks.json`) e o app Windows é um *wrapper* Electron que abre a
produção em janela nativa. Assim, cada deploy na Vercel atualiza todos os clientes ao mesmo tempo.

## Modelo de dados

Tabelas principais (Postgres, todas com RLS):

| Tabela | Papel |
|---|---|
| `tenants` | Escritório/empresa. Guarda plano, status da assinatura e limite de assentos |
| `users` | Perfil do técnico (registro profissional, conselho, ART, assinatura) e papel |
| `vistorias` | Cliente, endereço, tipo, status, conclusão, modelo de laudo e relatório gerado |
| `ambientes` | Cômodos/áreas de cada vistoria, com texto descritivo |
| `medias` | Fotos, vídeos e áudios por ambiente, com legenda, ordem e transcrição |
| `checklist_itens` | Itens de norma ABNT marcados na vistoria |
| `modelos_relatorio` | Biblioteca de modelos DOCX/PDF por usuário |
| `signup_requests` | Pedidos de acesso aguardando aprovação |
| `plans` | Planos Solo/Team com IDs de preço mensal e anual do Stripe |

Buckets de Storage separados para mídias, ART/RRT e assinaturas, cada um com políticas de escrita
restritas ao dono.

### Isolamento multi-tenant

- Cada usuário pertence a um `tenant`. As políticas RLS filtram leitura e escrita pelo tenant do
  usuário autenticado.
- Papéis: `platform_admin` (operador do SaaS), `tenant_admin` (dono do escritório) e `user` (técnico).
- Um *trigger* (`users_protect_role`) impede que o próprio cliente altere o papel. Só o `service_role`
  pode promover usuários. Convidados sempre entram como `user`.

## Offline-first

A coleta em campo não pode depender de sinal de celular. A estratégia de gravação é decidida por uma
função pura e testada:

| Modo offline | Online | Estratégia |
|---|---|---|
| desligado | sim | upload direto |
| desligado | não | bloqueado (avisa o usuário) |
| ligado | sim | enfileira no IndexedDB, devolve item local e sincroniza em segundo plano |
| ligado | não | só enfileira |

Com o modo offline ligado, a interface nunca espera pela rede. Os IDs locais das mídias são mapeados
para os IDs definitivos após a sincronização, e há reparo automático de descrições que referenciavam
IDs locais.

O manifest do PWA registra um **share target**, então fotos, vídeos e áudios recebidos no WhatsApp
podem ser enviados para uma vistoria pelo menu "Compartilhar" do Android.

## Pipeline de IA

As rotas `/api/transcrever`, `/api/gerar-relatorio` e `/api/revisar-relatorio` chamam webhooks do n8n:

1. **Transcrever:** baixa cada áudio do Storage, transcreve com Whisper, junta as transcrições na
   ordem de gravação e devolve o texto completo ou um resumo (GPT-4o-mini).
2. **Gerar laudo:** baixa o modelo de laudo escolhido, extrai o texto, classifica o conteúdo da vistoria
   pelas seções do modelo e redige o laudo com GPT-4o. A resposta traz marcadores
   (`[[MIDIA:foto|url|legenda]]`) que o app troca por imagens reais no documento.
3. **Revisar:** aplica uma instrução do técnico sobre o laudo já gerado ("deixe mais formal",
   "inclua a NBR 15575").

Manter a orquestração no n8n permite ajustar prompts e modelos sem novo deploy.

## Documento final

- **DOCX** gerado com a biblioteca `docx` e **PDF** a partir do mesmo HTML intermediário.
- Fotos baixadas do Storage e normalizadas com `sharp` (rotação EXIF, redimensionamento, JPEG).
- ART/RRT em PDF rasterizada (`pdfjs-dist` + `@napi-rs/canvas`) e anexada como imagem.
- Assinaturas dinâmicas: blocos por cargo (técnico responsável, proprietário, acompanhante…),
  desenhadas no app ou enviadas como imagem.

## Cobrança

- Planos **Solo** (1 assento) e **Team** (até 4 assentos), com preço mensal ou anual.
- Stripe Checkout cria a assinatura. O webhook provisiona o tenant e atualiza o status.
- `past_due` mantém o acesso (período de carência). `canceled` e `unpaid` bloqueiam o app e
  redirecionam para `/acesso-bloqueado`.
- Customer Portal para troca de cartão, upgrade para Team e cancelamento.

## Testes

- 20 suítes unitárias com `node:test` cobrindo geração de laudo, assinaturas, export, estratégia offline,
  mapeamento de IDs, links de autenticação, planos e provisionamento Stripe.
- Scripts E2E contra o ambiente real: autenticação completa (cadastro, convite, recuperação de senha),
  isolamento multi-tenant e os workflows de IA.
