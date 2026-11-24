# Fluxo360 — sistema web integrado

Projeto inicial de plataforma web para consolidar as planilhas **DASHBOARD 2026.xlsx** e **SISTEMA DE CONTROLE DE ESTOQUE 2.0 _ VERSÃO 2025 PREENCHIDO.xlsx**.

## O que já está preparado

- Interface responsiva para computador e celular.
- Login por e-mail e senha via Supabase Auth.
- Dashboard com indicadores de faturamento, lucro real, produtos e alertas de estoque.
- Filtros globais por ano e mês para consultar históricos de diferentes períodos.
- Importação de vários arquivos de dashboard anuais na mesma operação; cada indicador financeiro é gravado com ano e mês, sem sobrescrever outros anos.
- Módulos de produtos, fornecedores, entradas, saídas, compras, financeiro e comissões.
- Leitura de arquivos Excel no navegador e prévia antes da importação.
- Integração de produtos, fornecedores, movimentações, indicadores financeiros e comissões.
- Importação repetida com chaves únicas e `upsert` para reduzir duplicações.
- Banco PostgreSQL no Supabase com Row Level Security (RLS), isolamento por espaço de trabalho e dois acessos individuais compartilhando a mesma base.

## Importante sobre a importação

A planilha de estoque de 2025 tem milhares de linhas de movimentações e a planilha de dashboard de 2026 possui dados financeiros e comissões. O importador identifica o ano pelo título da aba/arquivo e permite importar dashboards de anos diferentes em cargas separadas ou em conjunto. Os arquivos originais contêm fórmulas quebradas/referências externas e registros sem data ou código em algumas linhas. O importador lê os valores calculados disponíveis, ignora linhas sem produto/código ou quantidade e preserva as movimentações importáveis. Faça a conferência dos totais após a carga antes de usar o sistema como fonte contábil.

O importador atual mapeia o cadastro de produtos e fornecedores, entradas/saídas, a tabela mensal `DADOS - Dashboard` e a aba `COMISSAO`. Campos complementares e fórmulas específicas do arquivo (por exemplo, composição detalhada do caixa, taxas de marketplace e dashboards STE & TAMI) precisam de uma etapa de mapeamento/validação adicional antes de serem considerados automatizados.

## Configuração local

1. Instale Node.js LTS.
2. Crie um projeto em https://supabase.com/.
3. No Supabase, abra **SQL Editor** e execute `supabase/schema.sql`.
4. Em **Authentication → Providers**, habilite Email. Crie o primeiro usuário (proprietário) e depois convide/crie o segundo usuário.
5. Para o segundo usuário compartilhar a mesma empresa, copie os UUIDs dos dois usuários em **Authentication → Users** e execute no SQL Editor (substitua os UUIDs):

   ```sql
   -- Adiciona o segundo login ao espaço de trabalho do proprietário.
   insert into public.workspace_members(workspace_id,user_id,role)
   select owner_member.workspace_id, 'UUID_DO_SEGUNDO_USUARIO', 'member'
   from public.workspace_members owner_member
   where owner_member.user_id = 'UUID_DO_PRIMEIRO_USUARIO' and owner_member.role = 'owner'
   on conflict do nothing;

   -- Remove o espaço de trabalho pessoal automático do segundo login,
   -- para ele não ficar associado a duas empresas.
   delete from public.workspace_members
   where user_id = 'UUID_DO_SEGUNDO_USUARIO'
     and workspace_id not in (
       select workspace_id from public.workspace_members
       where user_id = 'UUID_DO_PRIMEIRO_USUARIO' and role = 'owner'
     );
   ```
6. Copie `.env.example` para `.env` e preencha:

   ```env
   VITE_SUPABASE_URL=https://SEU-PROJETO.supabase.co
   VITE_SUPABASE_ANON_KEY=SUA_CHAVE_PUBLICA_ANON
   ```

7. Instale e execute:

   ```bash
   npm install
   npm run dev
   ```

8. Acesse o endereço local indicado pelo Vite.
9. Faça login e use **Importar planilhas** para selecionar os dois arquivos.

## Publicação em domínio próprio

Hospede o front-end em Vercel, Netlify ou serviço equivalente e conecte seu domínio (por exemplo, `gestao.seudominio.com.br`). Configure as mesmas variáveis de ambiente na hospedagem. O Supabase fica responsável pela autenticação e banco de dados.

Os dados financeiros são identificados por ano e mês. O filtro global permite consultar cada ano e mês; os movimentos de estoque são filtrados pela data original. O saldo atual do estoque continua considerando todo o histórico de entradas e saídas.

Antes de colocar em produção:

- Use HTTPS (normalmente automático na hospedagem gerenciada).
- Nunca exponha `service_role` no navegador; somente a chave pública com RLS.
- Desative cadastro público se o sistema for de uso interno e crie os usuários manualmente.
- Configure recuperação de senha, MFA quando disponível, política de senhas e backup do banco.
- Restrinja as URLs de redirecionamento do Supabase ao domínio de produção e localhost.
- Teste acesso com duas contas diferentes para validar o isolamento.
- Faça uma cópia dos Excel originais e valide saldos, custos e faturamento antes de substituir a rotina atual.

## Limitações desta primeira versão

É um starter funcional de interface e importação, não uma implantação hospedada. Não inclui ainda edição manual de produtos/movimentos, auditoria detalhada, perfis de permissões por função, exportação de relatórios ou integração automática com marketplaces/fornecedores. Essas etapas devem ser implementadas e testadas antes da operação definitiva.
