# ShivaOS Kernel

Fedora vanilla kernel + BORE scheduler + BBR3 TCP par défaut.

## Ce qu'on ajoute au vanilla Fedora

### Patch BORE (Burst-Oriented Response Enhancer)
- Source : [firelzrd/bore-scheduler](https://github.com/firelzrd/bore-scheduler) — branche `patches/stable/linux-7.0-bore`
- Le scheduler EEVDF de Linux mesure la "burstiness" de chaque tâche
- Les tâches interactives (jeux, audio, UI) sont prioritaires sur les tâches batch
- Résultat : latence gaming réduite, moins de stuttering sous charge

### Configs ajoutées dans `kernel-x86_64-fedora.config`
```
CONFIG_SCHED_BORE=y
CONFIG_MIN_BASE_SLICE_NS=2000000
CONFIG_DEFAULT_BBR=y
# CONFIG_DEFAULT_CUBIC is not set
CONFIG_TCP_CONG_BBR=y
```

### Ce qui est déjà dans vanilla Fedora (pas besoin de patcher)
- `CONFIG_HZ_1000=y`
- `CONFIG_PREEMPT_DYNAMIC=y`
- `CONFIG_NTSYNC=y`

## Version actuelle
- **Kernel** : `7.0.8-200.shivaos1.fc44`
- **COPR** : [freuja/ShivaOs](https://copr.fedorainfracloud.org/coprs/freuja/ShivaOs/)
- **Base** : Fedora 44

## Build

```bash
# Télécharger le SRPM Fedora vanilla depuis Koji
# Extraire kernel-x86_64-fedora.config + patch-7.0-redhat.patch + linux-*.tar.xz
# Copier dans ~/rpmbuild/SOURCES/ + appliquer les configs ShivaOS
rpmbuild -bs kernel.spec \
  --define "_topdir ~/rpmbuild" \
  --without debug --without debuginfo --without kabichk

# Upload COPR
curl -s -X POST \
  -u "LOGIN:TOKEN" \
  -F "projectname=ShivaOs" -F "ownername=freuja" \
  -F "pkgs=@kernel-7.0.8-200.shivaos1.fc44.src.rpm" \
  "https://copr.fedorainfracloud.org/api_3/build/create/upload"
```

## Intégration dans ShivaOS Atomic (Containerfile)

```dockerfile
RUN BASE="https://download.copr.fedorainfracloud.org/results/freuja/ShivaOs/fedora-44-x86_64/BUILD_ID-kernel" && \
    rpm-ostree override replace \
    "${BASE}/kernel-7.0.8-200.shivaos1.fc44.x86_64.rpm" \
    "${BASE}/kernel-core-7.0.8-200.shivaos1.fc44.x86_64.rpm" \
    "${BASE}/kernel-modules-7.0.8-200.shivaos1.fc44.x86_64.rpm" \
    "${BASE}/kernel-modules-core-7.0.8-200.shivaos1.fc44.x86_64.rpm" \
    "${BASE}/kernel-modules-extra-7.0.8-200.shivaos1.fc44.x86_64.rpm" \
    "${BASE}/kernel-devel-7.0.8-200.shivaos1.fc44.x86_64.rpm" \
    "${BASE}/kernel-devel-matched-7.0.8-200.shivaos1.fc44.x86_64.rpm" && \
    ostree container commit
```

⚠️ `kernel-devel-matched` **obligatoire** dans l'override — sinon conflit dépendance avec kinoite:44 base.
