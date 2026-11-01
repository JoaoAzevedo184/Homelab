# 6. Integração com o GitHub

- Chave SSH ed25519 gerada no servidor e cadastrada na conta (Settings → SSH keys).
- Considerei usar Deploy key, mas ela vale para um repositório só; como o servidor vai rodar vários projetos, a chave na conta foi a escolha.
- Mitigação: SSH só com chave, UFW, fail2ban e acesso externo apenas via Tailscale. Em caso de problema, revogar a chave no GitHub.
- Projetos ficam em /opt/apps/<projeto>; infraestrutura em /opt/homelab.
- Futuro: GitHub Actions + GHCR para o servidor só baixar imagens.