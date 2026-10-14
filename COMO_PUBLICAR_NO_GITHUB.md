# Passo a passo — publicar no GitHub

## Opção 1 — Pelo navegador (mais simples)

1. Entre em https://github.com e faça login.
2. Clique em **+** (canto superior direito) → **New repository**.
3. Nomeie o repositório `saga-ecommerce-architecture`.
4. Selecione **Public** (ou **Private**, conforme a orientação do professor).
5. **Não** marque `Add a README file`, pois este projeto já possui `README.md`.
6. Clique em **Create repository**.
7. Na página do repositório vazio, clique em **uploading an existing file** (ou **Add file → Upload files**, se exibido).
8. **Extraia o ZIP primeiro.** Abra a pasta `saga-ecommerce-architecture` e arraste **todos os arquivos e pastas de dentro dela** para a área de upload. Confira se `README.md` ficou na raiz e `diagrams/` e `docs/` estão no nível correto.
9. Na área **Commit changes**, digite `Adiciona blueprint de arquitetura Saga` e confirme.
10. Atualize a página. O README e os diagramas devem aparecer automaticamente. Copie o endereço do repositório e entregue o link no ambiente da atividade.

**Se o navegador não aceitar arrastar pastas:** use a opção 2, que preserva a estrutura completa com segurança.

## Opção 2 — Pelo VS Code + Git

1. Instale Git em https://git-scm.com/downloads e VS Code em https://code.visualstudio.com/.
2. Extraia o ZIP, abra o VS Code e escolha **File → Open Folder**; selecione `saga-ecommerce-architecture`.
3. Crie no GitHub um repositório **vazio** chamado `saga-ecommerce-architecture`, sem README automático.
4. Abra **Terminal → New Terminal** no VS Code e execute, uma linha por vez:

```bash
git init
git add .
git commit -m "Adiciona blueprint de arquitetura Saga"
git branch -M main
git remote add origin https://github.com/SEU-USUARIO/saga-ecommerce-architecture.git
git push -u origin main
```

5. Substitua `SEU-USUARIO` pelo seu usuário real do GitHub. Quando solicitado, faça login usando o fluxo de autenticação do Git Credential Manager ou outro método autorizado pelo GitHub (não use a senha da conta como senha Git).
6. Abra a URL do repositório para conferir os arquivos.

Se o Git reclamar de identidade, configure antes do commit:

```bash
git config --global user.name "Seu Nome"
git config --global user.email "seu-email@example.com"
```

## Checklist antes de entregar

- [ ] `README.md` na raiz e imagens carregando no GitHub.
- [ ] Três arquivos PNG em `diagrams/`.
- [ ] Arquivos `.dot` editáveis e `.svg` vetoriais em `diagrams/`.
- [ ] `docs/decisoes-arquiteturais.md` presente.
- [ ] O repositório está visível para o professor.
- [ ] Link do repositório enviado no AVA.
