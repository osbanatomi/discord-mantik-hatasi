# Discord Server Boost Transfer — İş Mantığı Hatası

Discord’un sunucu takviyesi aktarma özelliğini incelerken, aktarım sırasında belirli bir HTTP isteğinin engellenmesiyle takviyenin kaynak sunucuda kalırken hedef sunucuya da uygulanabildiğini gözlemledim.

Bu çalışma, istemcinin gönderdiği isteklerin sırasını değiştirmenin uygulamanın iş kuralları üzerindeki etkisini inceliyor.

## Beklenen Davranış

Bir sunucu takviyesi başka bir sunucuya aktarıldığında, kaynak sunucudaki takviye kaldırılmalı ve hedef sunucuya uygulanmalıdır. Aynı takviye hakkının eş zamanlı olarak iki farklı sunucuda kullanılmaması gerekir.

## Keşif Süreci

Takviye aktarımı sırasında istemci ile Discord API arasındaki HTTP trafiğini bir proxy aracıyla (Burp Suite) inceledim. Gözlemlediğim akışta istemci sırasıyla:

1. Kaynak sunucudaki takviye kaydını kaldırmak için `DELETE` isteği gönderiyordu.
2. Kullanıcının takviye slotlarını `GET` isteğiyle sorguluyordu.
3. Seçilen takviyeyi hedef sunucuya uygulamak için `PUT` isteği gönderiyordu.

Bu işlemlerin ayrı isteklerle gerçekleşmesi şu soruyu doğurdu:

> Kaynak sunucudaki takviyeyi kaldıran istek sunucuya ulaşmazsa, hedef sunucuya takviye uygulama işlemi yine de kabul edilir mi?

Bu varsayımı test etmek için aktarım sırasında yalnızca `DELETE` isteğini engelledim ve sonraki isteklerin devam etmesine izin verdim.

## Test Edilen İstek Akışı

Kaynak sunucudaki takviye kaydını kaldıran istek:

```http
DELETE /api/v9/guilds/{source_guild_id}/premium/subscriptions/{subscription_id}
Host: discord.com
```

Bu isteği proxy üzerinde **drop** ederek sunucuya ulaşmasını engelledim.

Ardından, kullanıcının takviye slotlarını sorgulayan `GET` isteğine ve hedef sunucuya takviye uygulayan aşağıdaki isteğe izin verdim:

```http
PUT /api/v9/guilds/{target_guild_id}/premium/subscriptions
Host: discord.com
```

Buradaki kimlikler yer tutucudur. Silme isteğindeki abonelik kaydı kimliği ile takviyeyi uygularken kullanılan slot kimliği ayrı alanlar olarak değerlendirilmelidir.

## Gözlemlenen Sonuç

Silme isteği engellenmesine rağmen hedef sunucuya takviye uygulama işlemi kabul edildi. Test sırasında:

* Kaynak sunucudaki mevcut takviye kaldırılmadı.
* Hedef sunucuya takviye uygulandı.
* Aynı takviye hakkı iki farklı sunucuda etkili görünür hâle geldi.

Bu davranış, takviye aktarımında korunması gereken “bir takviye hakkının aynı anda yalnızca bir sunucuda kullanılması” kuralıyla çelişiyordu.

## Teknik Değerlendirme

Gözlemlediğim davranış, kaynak sunucudan kaldırma ve hedef sunucuya uygulama işlemleri arasında yeterli sunucu tarafı doğrulama bulunmadığıydı.

Olası açıklama, hedef sunucuya takviye uygulayan işlemin, önceki kaldırma işleminin başarıyla tamamlandığını zorunlu olarak doğrulamamasıdır. Ancak sunucu tarafındaki uygulamaya erişimim olmadığı için kesin kök nedeni doğrulamadım.

Bu testte eş zamanlı istek göndermedim; tek bir isteği engellemek davranışı tetiklemek için yeterliydi. Dolayısıyla bulguyu doğrudan bir **race condition** olarak sınıflandırmak yerine, **iş mantığı ve durum tutarlılığı hatası** olarak değerlendiriyorum.

## Etki ve Sınırlar

Gözlemlenen etki, mevcut bir takviye hakkının aktarım sonrasında birden fazla sunucuda etkili görünmesiydi. Bu durum, ücretli bir özelliğin kullanım kurallarının aşılmasına yol açabilir.

Bununla birlikte bu test, yeni bir takviye slotunun gerçekten oluşturulduğunu, durumun kalıcı olduğunu veya sınırsız takviye üretilebildiğini tek başına kanıtlamaz. Kalıcılık ve sonradan gerçekleşebilecek otomatik düzeltmeler ayrıca değerlendirilmelidir.

## Çıkarım

Bu araştırma, güvenlik açısından önemli iş kurallarının istemcinin bütün istekleri doğru sırayla göndermesine bağlı olmaması gerektiğini gösteriyor.

Takviye aktarımı gibi işlemlerde sunucu, mevcut atamayı doğrulamalı ve kaynak sunucudan kaldırma ile hedef sunucuya uygulama adımlarının tutarlı biçimde tamamlanmasını sağlamalıdır.
