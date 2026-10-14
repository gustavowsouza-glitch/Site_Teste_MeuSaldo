# Meu saldo — Finanças pessoais

Site em português para bancos, cartões, receitas, despesas, transferências, contas a pagar e compromissos mensais. Next.js, TypeScript, Tailwind CSS, Supabase (Auth + PostgreSQL) e hospedagem na Vercel.

## Rodar no computador

Requisitos: Node.js 22 LTS ou superior e pnpm 11. As versões resolvidas estão em `pnpm-lock.yaml`.

```powershell
pnpm install --frozen-lockfile
pnpm dev
```

Abra http://127.0.0.1:3000. Sem variáveis do Supabase, o site funciona em **modo local**: começa sem bancos ou valores fictícios, com dados salvos no navegador, sem login e sem sincronização. Cadastre o banco e informe o saldo disponível para começar; as movimentações seguintes ajustam esse saldo. Dados já salvos são preservados. “Começar do zero” apaga todos os dados locais somente após confirmação. Esses dados nunca são importados para sua conta com login. Com Supabase, cada funcionário deve ter seu próprio usuário; os dados são separados por usuário, não compartilhados entre funcionários.

## Configurar o Supabase

1. Crie um projeto em [Supabase](https://supabase.com/dashboard). No SQL Editor, execute o conteúdo de `supabase/migrations/202610140001_finance.sql` uma vez. Alternativamente, use `supabase link` e `supabase db push` com a CLI oficial. Não execute novamente a migração aplicada; acrescente novas migrações para mudanças futuras.
2. No Supabase Auth, habilite o provedor de senha e **Allow new users to sign up**. Para cadastro somente por nome de usuário, desabilite **Confirm email**. O aplicativo usa internamente `usuario@usuarios.meusaldo.invalid` como identificador reservado do provedor; isso não é uma caixa postal. As senhas continuam sendo gerenciadas pelo Supabase Auth.
3. Os funcionários podem clicar em **Criar conta** na tela de login e informar somente nome de usuário e senha. Os nomes têm 3 a 32 caracteres (letras sem acentos, números, `_` e `-`), não diferenciam maiúsculas e são únicos. O funcionário não precisa ler este README; a configuração do serviço é feita uma vez pelo responsável pelo site. Cada usuário acessa somente seus próprios dados. Contas antigas por email continuam podendo entrar por email. Contas sem email não têm recuperação automática por email: o responsável precisa verificar a identidade da pessoa e redefinir a senha no Supabase. A alteração dessa configuração também afeta futuros cadastros diretos por email no mesmo projeto.
4. Copie `.env.example` para `.env.local` e preencha `NEXT_PUBLIC_SUPABASE_URL` e `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY` com a URL e a chave publicável do projeto (a chave `anon` legada também funciona). A chave de serviço não é necessária para o aplicativo e nunca deve receber prefixo `NEXT_PUBLIC_`.
5. Em **Authentication → URL Configuration**, defina o Site URL e autorize as URLs de redirecionamento da aplicação: localmente, `http://127.0.0.1:3000/auth/confirm` e `http://127.0.0.1:3000/nova-senha`; em produção, os mesmos caminhos no domínio HTTPS publicado.
6. Em **Email Templates → Reset Password**, configure o link assim:

```html
<a href="{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=recovery">Criar nova senha</a>
```

Para convite, em **Invite user**, use:

```html
<a href="{{ .SiteURL }}/auth/confirm?token_hash={{ .TokenHash }}&type=invite">Definir minha senha</a>
```

Para testar recuperação local, o Site URL deve apontar para a aplicação local. Use um projeto separado para desenvolvimento se a produção já estiver ativa. Configure SMTP próprio no Supabase para entregar mensagens em produção a destinatários permitidos; o serviço padrão tem restrições e limites. A recuperação não revela se um email tem cadastro.

Reinicie `pnpm dev` após editar o ambiente. O acesso real é protegido no servidor; cada consulta do banco usa Row Level Security. As operações de escrita passam por funções autenticadas com validação de propriedade. Referências compostas impedem vincular bancos, cartões ou categorias de outro usuário.

## Regras de uso

- Valores são armazenados em centavos; informe `1.250,50`. O saldo inicial de um banco pode ser negativo.
- Saldo atual = saldo inicial + receitas − despesas − transferências enviadas + transferências recebidas, inclusive contas arquivadas. Arquivamento preserva histórico; contas arquivadas não aceitam novas movimentações ou pagamentos.
- O resumo mensal considera somente receitas e despesas realizadas. Transferências não alteram o resultado. Movimentações realizadas não aceitam datas futuras; registre compromissos futuros em contas a pagar.
- Contas pendentes não alteram saldos. “Pagar” informa a conta de origem e a data; pagamento e despesa são gravados na mesma transação. Duas solicitações simultâneas não geram duas despesas. Desfazer remove a despesa e reabre a conta.
- Movimentações de pagamento são gerenciadas pela conta paga. Para mudar valor, descrição ou categoria, desfaça o pagamento, edite a conta e pague novamente.
- Cartões guardam somente informações básicas e quatro dígitos finais, nunca número completo ou CVV. Limite informativo; cadastre faturas como contas a pagar para registrar o desembolso uma única vez.
- Em **Contas a pagar → Contas mensais**, crie uma regra mensal. Ao abrir/atualizar, o sistema gera do primeiro mês até o mês atual, inclusive os meses faltantes, sem duplicação. Uma regra iniciada no futuro gera seu primeiro lançamento quando esse mês chegar. Dias 29–31 são ajustados ao último dia disponível.
- Cada conta gerada pode ser editada/excluída independentemente. Excluir uma ocorrência não a recria. Editar a regra preserva ocorrências já geradas. Pausar interrompe novas gerações; retomar preenche os meses ainda não gerados com os valores atuais da regra. O primeiro mês não é alterável após criar a regra.
- Datas usam o calendário de America/Bahia. Contas vencidas são pendentes com vencimento anterior a hoje; contas de hoje não são atrasadas.
- Dados são atualizados ao abrir, ao voltar para a aba e a cada minuto. O acesso em diferentes dispositivos consulta a mesma conta no Supabase. Sem internet, um salvamento apresenta erro; não há fila offline.
- Categorias em uso não podem ser excluídas; renomeie ou edite seus vínculos. A exportação CSV inclui os filtros atuais, usa separador `;`, acentos e proteção contra fórmulas de planilha. Não é um backup completo do banco; use os recursos de backup do Supabase para isso.

## Verificar

```powershell
pnpm typecheck
pnpm test
pnpm test:db
pnpm build
```

`test:db` executa a migração em PostgreSQL em memória via PGlite com dois usuários e papéis SQL. Verifica RLS, referências entre usuários, permissões, pagamentos repetidos, reversão, atomicidade, arquivos e recorrências. Simula somente `auth.uid()`; não substitui validar Supabase Auth e entrega de emails no projeto hospedado.

Validação real após configurar: entrar; cadastrar banco e cartão; lançar receita; criar conta mensal; pagar; conferir saldo e despesa única; desfazer; recarregar; entrar pelo celular e verificar os mesmos dados; sair; recuperar senha. Testar também com outro usuário que não deve enxergar seus registros.

## Publicar na Vercel

1. Envie o projeto a um repositório Git privado e importe-o na [Vercel](https://vercel.com/new), usando o preset Next.js. Como alternativa, execute `pnpm dlx vercel` neste diretório e siga o login da sua conta.
2. Configure as duas variáveis do Supabase no ambiente Production (e em Preview somente se desejar testar com banco separado). O build usa `pnpm build`. Não publique dados pessoais no modo demonstração.
3. Execute o deploy e, no Supabase, ajuste Site URL, redirecionamentos e modelos de email para o domínio definitivo. Redeploy após mudar variáveis.
4. Faça a validação real descrita acima antes de começar a inserir seus dados. Confira os logs da Vercel e os logs de Auth/Postgres no Supabase caso haja falhas. Custos e limites seguem os planos escolhidos nos provedores.

Nenhum projeto hospedado ou usuário real é criado automaticamente pelo código. Publicação e teste de login/email/sincronização exigem as contas e a configuração dos serviços.
#   S i t e _ T e s t e _ M e u S a l d o  
 