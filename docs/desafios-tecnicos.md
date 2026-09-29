# Desafios técnicos

[← Voltar ao README](../README.md)

Problemas reais encontrados durante o desenvolvimento e como foram resolvidos.

## Fotos de celular saíam como link no Word

**Sintoma:** no laudo exportado, algumas fotos apareciam como texto "Foto: https://…" em vez de imagem.

**Causa:** a rota que embute mídias rejeitava arquivos acima de 8 MB *antes* de redimensionar. Fotos de
celulares modernos passam disso por pouco.

**Solução:** o teto de download subiu para 25 MB e toda imagem passa por normalização com `sharp` antes
de ser embutida. O fallback em link só acontece se a normalização falhar.

## Convidado entrava como administrador

**Sintoma:** ao abrir o link de convite no mesmo navegador em que o admin estava logado, o convidado via
as telas de administração.

**Causa:** a sessão anterior persistia no callback de autenticação, e a política RLS permitia que o
próprio cliente alterasse o campo `role`.

**Solução:** logout local antes de validar o token do convite, *trigger* no banco que impede alteração
de papel fora do `service_role`, bloqueio de `/admin/*` no middleware e teste E2E que confere o papel.

## Links de convite e recuperação abriam o lugar errado

**Sintoma:** o link do e-mail abria o dashboard (sem pedir nova senha) ou apontava para `localhost`,
falhando no celular.

**Causa:** o Auth self-hosted substituía o `redirect_to` por uma URL padrão e o callback usava o
dashboard como destino quando faltava o parâmetro `next`.

**Solução:** os e-mails passaram a ser enviados pelo próprio app (Resend) com link contendo `token_hash`.
O callback valida o token e força `/redefinir-senha` para convites e recuperações. Uma coluna
`password_setup_required`, lida pelo middleware, garante que o usuário defina a senha antes de usar o app.

## Erro "Falha ao convidar: {}"

**Causa:** sem SMTP configurado, o Auth retornava HTTP 500 e o SDK tratava a resposta como erro de rede
antes de ler o JSON. A mensagem chegava vazia.

**Solução:** o convite passou a usar `generateLink` (cria o usuário sem depender de SMTP) e o e-mail é
enviado pelo Resend, com mensagens de erro legíveis.

## Migração para multi-tenant

**Desafio:** introduzir `tenant_id` e novos papéis em um banco já em uso.

**Problemas:** o *trigger* de proteção de papel revertia o *backfill* porque a execução administrativa
não tinha JWT de `service_role`. Além disso, a função que cria o perfil de novos usuários inseria o tenant
com uma chave estrangeira para um usuário que ainda não existia.

**Solução:** desabilitar o *trigger* só durante o *backfill* e criar o tenant sem dono, ligando o dono
depois que o perfil existe.

## Empacotamento Android no Windows

**Desafio:** gerar o app Android (TWA) com Bubblewrap em ambiente Windows.

**Problemas:** incompatibilidades entre o layout novo do Android SDK e o que o Bubblewrap valida, versões
específicas de build-tools, prompts interativos e diferenças de execução de `.bat`.

**Solução:** um script (`npm run android:build`) que prepara JDK e SDK, cria os links de diretório
esperados, responde aos prompts e gera o AAB para a Play Store. O `assetlinks.json` é publicado junto
para o app abrir em tela cheia.

## App Windows não abria

**Sintoma:** clicar no atalho não mostrava nenhuma janela.

**Causa:** a janela Electron só aparecia no evento `ready-to-show`, que às vezes não disparava com rede
lenta. O processo ficava vivo e invisível, e o *single instance lock* fazia os cliques seguintes apenas
focarem essa janela invisível.

**Solução:** a janela passa a ser revelada no `ready-to-show` ou, no máximo, após 2,5 s. Falhas de carregamento (`did-fail-load`) são tratadas, e uma segunda instância mostra ou recria a janela em vez de só focar.
