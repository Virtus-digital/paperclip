# Virtus-digital/paperclip

[paperclipai/paperclip](https://github.com/paperclipai/paperclip) deposunun
Virtus fork'u. Tek işi, fra kümesine dağıtılan **Clipbox** imajını
(`ghcr.io/virtus-digital/paperclip`) **kontrollü** sürümlerden derlemek.
Dağıtım `Virtus-digital/deploy` → `platform-fra/paperclip/` altında.

Son güncelleme: 2026-10-01

## Bu dalda ne var, ne yok

Varsayılan dal `virtus` upstream koduyla **ortak geçmiş taşımıyor**. İçinde
yalnız bu dosya ve `.github/workflows/imaj.yml` var. Upstream kodu bu depoda
yalnız **aynalanan sürüm etiketleri** olarak duruyor:

```
refs/tags/upstream/v2026.916.1   → upstream'in v2026.916.1 etiketiyle AYNI commit
```

Bilerek böyle kuruldu:

| Ne | Neden |
|---|---|
| `master` dalı yok | Upstream'in workflow'ları `push: branches: master` ile tetikleniyor. Dal yoksa tetiklenecek bir şey de yok. GitHub'ın "Sync fork" düğmesi de bu yüzden işe yaramıyor; güncelleme elle ve bilerek yapılıyor. |
| Etiketler `upstream/` önekli | Upstream'in `docker.yml`'i `v*`, `nightly/v*`, `beta/v*` etiketlerinde tetikleniyor. Aynı adla aynalanan bir etiket o workflow'u bu fork'ta koşturur ve imajı yanlış yoldan yayımlar. |
| Varsayılan dal upstream dosyası taşımıyor | Zamanlanmış upstream işleri (`schedule`) yalnız varsayılan daldan koşar. |

## Derleme nerede koşuyor

Virtus ARC'de, GitHub'ın kendi runner'larında değil. Bu depo public olduğu
için org'un ana ARC grubu (`Default`) onu çalıştırmıyor. Derleme kendi
grubunda (`paperclip-imaj`) ve kendi scale set'inde koşuyor:
`Virtus-digital/deploy` → `platform-fra/arc-runners-paperclip/`.

Grup yalnız `.github/workflows/imaj.yml@refs/heads/virtus`'a açık. Bu
dosyanın adı ya da dalı değişirse grubun `selected_workflows` listesi de
güncellenmeli, yoksa iş runner bulamaz:

```sh
gh api orgs/Virtus-digital/actions/runner-groups/3 --jq '.selected_workflows'
```

Dış katkıcıların PR'larında workflow'lar onaysız koşmuyor
(`fork-pr-contributor-approval: all_external_contributors`).

## Güncelleme yordamı

1. Upstream sürüm notlarını oku: <https://github.com/paperclipai/paperclip/releases>.
2. Etiketin commit'ini çöz. Etiket açıklamalıysa `^{}` satırı commit'tir:

   ```sh
   SURUM=v2026.916.1
   git ls-remote https://github.com/paperclipai/paperclip "refs/tags/$SURUM" "refs/tags/$SURUM^{}"
   ```

3. Etiketi `upstream/` önekiyle aynala (`<commit>` 2. adımın çıktısı):

   ```sh
   gh api -X POST repos/Virtus-digital/paperclip/git/refs \
     -f ref="refs/tags/upstream/$SURUM" -f sha=<commit>
   ```

   ⚠️ Önek olmadan itme. `git push --tags` de kullanma: upstream etiketlerini
   önek olmadan taşır.

4. İmajı derle:

   ```sh
   gh workflow run imaj.yml -R Virtus-digital/paperclip -f surum="$SURUM"
   ```

   Workflow önce iki kapıdan geçer. Birincisi: aynalanan etiket upstream'in
   aynı adlı etiketiyle aynı commit değilse durur. İkincisi: o imaj etiketi
   zaten yayımlanmışsa durur. Çıktının özeti `<imaj>:<sürüm>@<digest>`
   referansını basar.

5. Deploy PR'ı: `platform-fra/paperclip` manifestindeki imajı **etiket +
   digest** ile güncelle. Yükseltmeden önce CNPG'den anlık yedek al;
   migration'lar açılışta otomatik uygulanıyor, geri dönüş yolu DB'den geri
   yükleme.

## Yamalı sürüm gerekirse

Kaynak kapısı bugün yalnız upstream'le birebir aynı etiketi derliyor. Bir
yama gerekirse kapı **bilerek** değiştirilir (ör. `virtus/v<sürüm>` etiketi
ve kapının yeni bir dalı) ve değişiklik deploy tarafında kayda geçer. Kapı
sessizce atlanmaz.
