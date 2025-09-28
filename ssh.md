# Bonnes pratiques d’authentification SSH par clé publique/privée

## 1. Choisir les bons types de clés et protections
- **Algorithmes** :
  - **ed25519** en priorité (rapide, court, sûr).
  - **rsa** uniquement si legacy, avec **RSA 3072+** (4096 si possible).
  - Si possible, **FIDO2/U2F (hardware)** : `sk-ssh-ed25519@openssh.com` via clé matérielle (YubiKey, etc.).
- **Passphrase obligatoire** sur la clé privée. Utiliser un **agent** (ssh-agent, gnome-keyring, etc.).
- **Stockage** :
  - `~/.ssh` en **700**, clés privées en **600** ; ne partagez jamais la clé privée.
  - Sauvegarde chiffrée des clés privées (ex : coffre-fort mots de passe/secret manager).
- **Rotation** : cycle conseillé de 6–12 mois ou à chaque départ/changement d’appareil.

## 2. Génération & côté client
```bash
# clé logicielle ed25519
ssh-keygen -t ed25519 -a 100 -C "prenom.nom@exemple.com - laptop perso" -f ~/.ssh/id_ed25519

# clé matérielle FIDO2
ssh-keygen -t ed25519-sk -C "prenom.nom@exemple.com - yubikey" -f ~/.ssh/id_ed25519_sk
```

- Option `-a 100` renforce le KDF (bcrypt).
- Nommer clairement chaque clé (un fichier par appareil/usages).

Exemple `~/.ssh/config` :
```sshconfig
Host bastion
  HostName bastion.exemple.com
  User admin
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
  ForwardAgent no
  PubkeyAuthentication yes
```

## 3. Configuration côté serveur
```conf
# /etc/ssh/sshd_config
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
UsePAM yes
PermitRootLogin prohibit-password
PubkeyAcceptedKeyTypes ssh-ed25519,ssh-rsa
LogLevel VERBOSE
```

- `~/.ssh` **700**, `authorized_keys` **600**.
- Exemple d’options restrictives :
```text
from="10.0.0.0/24,192.0.2.10",command="/usr/local/sbin/limit.sh",no-agent-forwarding,no-port-forwarding,no-pty,no-user-rc ssh-ed25519 AAAAC3Nza... commentaire
```

## 4. Gestion multi-serveurs (CA SSH)
- **CA SSH** : meilleure pratique à l’échelle.
- Déployer la clé publique CA avec :
```conf
TrustedUserCAKeys /etc/ssh/ca_user.pub
```

## 5. MFA/2FA
- Privilégier les clés FIDO2 (touch required).
- Alternatives : PAM avec TOTP/Duo, ou bastion SSO.

## 6. Distribution, révocation, rotation
- Gérer `authorized_keys` via Ansible/Puppet.
- `RevokedKeys /etc/ssh/revoked_keys` possible.
- Une clé par appareil.

## 7. Politiques comptes & privilèges
- Interdire `root` en SSH ; utiliser `sudo`.
- Principe du moindre privilège.

## 8. Hygiène opérationnelle
- Inventaire des accès et audits réguliers.
- Mise à jour OpenSSH côté client/serveur.
- Activer le hash de `known_hosts`.

## 9. Exemples pratiques

### Ajouter une clé
```bash
ssh-copy-id -i ~/.ssh/id_ed25519 user@serveur
```

### Certificat utilisateur signé
```bash
ssh-keygen -s ~/.ssh/ca_user -I alice-2025Q3 -n alice -V +8h ~/.ssh/alice_id_ed25519.pub
```

## 10. Check-list rapide
- [ ] Clés **ed25519** avec **passphrase**.
- [ ] Droits stricts sur `~/.ssh` et `authorized_keys`.
- [ ] `PasswordAuthentication no`, `PermitRootLogin no`.
- [ ] Options restrictives dans `authorized_keys`.
- [ ] Bastion + `ProxyJump`, pas d’agent forwarding.
- [ ] CA SSH déployée si parc important.
- [ ] Rotation/révocation testées.
- [ ] Logs & alertes (fail2ban, SIEM).
- [ ] Procédure d’urgence (“break-glass”) testée.
