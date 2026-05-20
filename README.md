# ShivaOS Kernel

Fedora vanilla kernel + BORE scheduler + BBR3 TCP par défaut.

## Ce qu'on ajoute au vanilla Fedora

### Patch BORE (Burst-Oriented Response Enhancer)
- Source : [firelzrd/bore-scheduler](https://github.com/firelzrd/bore-scheduler) — branche `patches/stable/linux-7.0-bore`
- Le scheduler EEVDF de Linux mesure la "burstiness" de chaque tâche
- Les tâches interactives (jeux, audio, UI) sont prioritaires sur les tâches batch
- Résultat : latence gaming réduite, moins de stuttering sous charge

### Configs ajoutées dans `kernel-local` (s'applique à TOUTES les archis)

> ⚠️ Ne pas mettre dans `kernel-x86_64-fedora.config` — `process_configs.sh` vérifie toutes les
> architectures (aarch64, ppc64le, s390x, riscv64…) et fail si une config est absente d'une archi.

```
CONFIG_SCHED_BORE=y
CONFIG_MIN_BASE_SLICE_NS=2000000
CONFIG_TCP_CONG_BBR=y
CONFIG_DEFAULT_BBR=y
# CONFIG_DEFAULT_CUBIC is not set
```

### Ce qui est déjà dans vanilla Fedora (pas besoin de patcher)
- `CONFIG_HZ_1000=y`
- `CONFIG_PREEMPT_DYNAMIC=y`
- `CONFIG_NTSYNC=y`

## Version actuelle
- **Kernel** : `7.0.9-204.shivaos1.fc44`
- **COPR build** : [10488358](https://copr.fedorainfracloud.org/coprs/freuja/ShivaOs/build/10488358/) — succeeded
- **COPR** : [freuja/ShivaOs](https://copr.fedorainfracloud.org/coprs/freuja/ShivaOs/)
- **Base** : Fedora 44 vanilla 7.0.9

## Workflow rebase (nouvelle version upstream)

```bash
# 1. Télécharger le SRPM Fedora depuis Koji
#    https://kojipkgs.fedoraproject.org//packages/kernel/VERSION/RELEASE/src/

# 2. Extraire TOUS les fichiers (inclure patch-7.0-redhat.patch sinon build fail)
rpm2cpio kernel-*.src.rpm | cpio -id
cp -r * ~/rpmbuild/SOURCES/

# 3. Tester le patch BORE
patch -p1 -d linux-X.Y.Z --dry-run < bore-7.0.patch

# 4. Mettre à jour kernel.spec : specrpmversion, specversion, tarfile_release, kabiversion
# 5. Conserver kernel-local avec configs BORE/BBR (ne pas toucher)

# 6. Build SRPM
rpmbuild -bs kernel.spec --without debug --without debuginfo --without kabichk

# 7. Upload COPR
curl -s -X POST \
  -u "LOGIN:TOKEN" \
  -F "projectname=ShivaOs" -F "ownername=freuja" \
  -F "pkgs=@kernel-7.0.9-204.shivaos1.fc44.src.rpm" \
  "https://copr.fedorainfracloud.org/api_3/build/create/upload"
```

## Intégration dans ShivaOS Atomic (Containerfile)

```dockerfile
RUN BASE="https://download.copr.fedorainfracloud.org/results/freuja/ShivaOs/fedora-44-x86_64/10488358-kernel" && \
    rpm-ostree override replace \
    "${BASE}/kernel-7.0.9-204.shivaos1.fc44.x86_64.rpm" \
    "${BASE}/kernel-core-7.0.9-204.shivaos1.fc44.x86_64.rpm" \
    "${BASE}/kernel-modules-7.0.9-204.shivaos1.fc44.x86_64.rpm" \
    "${BASE}/kernel-modules-core-7.0.9-204.shivaos1.fc44.x86_64.rpm" \
    "${BASE}/kernel-modules-extra-7.0.9-204.shivaos1.fc44.x86_64.rpm" \
    "${BASE}/kernel-devel-7.0.9-204.shivaos1.fc44.x86_64.rpm" \
    "${BASE}/kernel-devel-matched-7.0.9-204.shivaos1.fc44.x86_64.rpm" && \
    ostree container commit
```

⚠️ `kernel-devel-matched` **obligatoire** dans l'override — sinon conflit dépendance avec kinoite:44 base.
