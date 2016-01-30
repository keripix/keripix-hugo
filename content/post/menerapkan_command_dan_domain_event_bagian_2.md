+++
date = "2016-01-27T14:43:58+07:00"
description = ""
title = "Menerapkan Command dan Domain Event bagian 2"

+++

![](https://cdn-images-1.medium.com/max/2000/1*LIYtUrLIm1oBr1537VgYPA.jpeg)

Pada pembahasan sebelumnya, kami membahas bagaimana Command diterapkan di codebase Dicoding. Pada artikel ini, kami akan mencoba untuk membahas penerapan Domain Events.

Namun sebelumnya, berikut adalah diagram yang menjelaskan secara sederhana hubungan antara Command, Command Bus dan Command Handler.

![](https://cdn-images-1.medium.com/max/800/1*ftFc9UoA-dOPtQaEGLudzw.png)

## Domain Event

Ketika logika bisnis sudah dijalankan oleh sistem, maka sebuah domain event dapat dibangkitkan. Domain Event berhubungan dengan kejadian penting dalam pelaksanaan sebuah use case.

Melanjutkan contoh `CreateUnpublishedChallenge` dari artikel sebelumnya, maka ketika Unpublished Challenge ini sudah terbuat, aplikasi dapat membangkitkan event `UnpublishedChallengeWasCreated`. Ketika sebuah domain event dibangkitkan, kita dapat menjalankan tugas-tugas tambahan yang berhubungan dengan event tadi.

Mari kita buat contoh sederhana untuk menggambarkan konsep di atas. Misalnya, kita memiliki use case berikut untuk proses memvalidasi sebuah Unpublished Challenge:

1. Admin meminta Unpublished Challenge untuk dipublikasikan.
2. Sistem memeriksa apakah Challenge sudah dibayar.
3. Sistem mengubah status Challenge menjadi Published.
4. Sistem mengabari Pemilik Challenge bahwa Challengenya sudah published.
5. Sistem menambahkan Challenge ke News Feeds.
6. Sistem mengirim tweets ke Twitter untuk mempublikasikan adanya Challenge baru.
7. Sistem melog kejadian di atas.


Ketika use case di atas diimplementasikan di sebuah Command Handler, kodenya bisa jadi seperti contoh berikut ini:

```
<?php

class PublishUnpublishedChallengeHandler
{
    public function handle(PublishUnpublishedChallenge $command)
    {
        try {
            
            DB::beginTransaction();
            
            $publishedChallenge = $this->challengePublicationContext->publishUnpublishedChallenge($command->challenge_id);
            
            $this->memberMailer->send($publishedChallenge->getChallengeOwner(), 'Congrats, your Challenge Has Been Published');
            
            $this->feedRepository->add($publishedChallenge);
            
            $this->tweetStream->add('A new challenge has been published');
            
            $this->eventLog->log($publishedChallenge);
            
            DB::commit();
            
        } catch (Exception $e) {
            
            DB::rollback();
            
            throw $e;
            
        }
        
        return new PublishUnpublishedChallengeResponse($publishedChallenge);
    }
}
```

Masalah dari contoh di atas adalah, bila ada salah satu kejadian di atas yang gagal, maka seluruh proses pembuatan Challenge akan menjadi batal. Hal ini karena seluruh tahapan di atas dijalankan di dalam sebuah database transaction.

Masalah kedua adalah, proses publishing di atas akan menunggu aktifitas-aktifitas yang membutuhkan waktu yang tidak sebentar. Misalnya mengirim email ke pemilik Challenge.

### Bagaimana Domain Event Dapat Membantu?

Yang perlu kita tentukan dari contoh use case di atas, adalah tugas mana yang utama dan tugas mana yang sekunder. Salah satu cara yang biasa saya lakukan adalah dengan menjawab pertanyaan ini:

> Dapatkah saya mentolerir bila tugas tersebut gagal?

Bila tugas `publishUnpublishedChallenge` gagal, dapatkah saya mentolerirnya? Tentu tidak. Ini adalah tugas utama. Bila dia gagal, seluruh proses harus gagal. Dan Admin yang mengirim request tersebut harus dikabari konteks kegagalannya.

Bagaimana dengan mengirim notifikasi kepada pemilik Challenge bahwa Challenge sudah terpublikasikan? Bila proses tersebut gagal, apakah kita dapat mentoleransinya? Bila saya ditanya pertanyaan ini, maka saya akan menjawab iya. Karena kejadian buruknya, kita dapat mengirim kabar ini secara manual.

Saya tidak berkata bahwa mengirim email ke pemilik Challenge tidak penting. Itu tetap penting. Tetapi seberapa jauh saya dapat mentoler kejadian gagal? Bila saya dapat mentolerirnya, maka ia masuk ke kategori tugas sekunder.

Saya juga akan memasukkan tugas menambahkan challenge ke feed, mengirim tweet ke twitter dan logging sebagai tugas sekunder.

Tentu saja mana yang masuk tugas utama dan tugas sekunder sangat tergantung pada kebutuhan bisnis. Bisa jadi di suatu konteks tertentu, mengirim email menjadi tugas utama. Bukan sekunder.

Setelah menentukan mana yang termasuk tugas utama, dan tugas sekunder, kode kita dapat berubah menjadi seperti contoh berikut ini:

```
<?php

class PublishUnpublishedChallengeHandler
{
    public function handle(PublishUnpublishedChallenge $command)
    {
        try {
            
            DB::beginTransaction();
            
            $publishedChallenge = $this->challengePublicationContext->publishUnpublishedChallenge($command->challenge_id);
            
            DB::commit();
            
        } catch (Exception $e) {
            
            DB::rollback();
            
            throw $e;
            
        }
        
        // Dispatch Domain Event
        $this->dispatchEventsFor($publishedChallenge);
        
        return new PublishUnpublishedChallengeResponse($publishedChallenge);
    }
}
```

Pada baris ke 23 pada contoh di atas, kita dapat melihat perintah `dispatchEventsFor`. Perintah ini akan mempublikasikan domain event yang telah dibangkitkan oleh `$publishedChallenge`. Dimanakan event-event ini dibangkitkan?

Dalam contoh di atas, domain event dapat dibangkitkan ketika proses publishUnpublishedChallenge dijalankan.

```
<?php

class ChallengePublicationContext
{
    public function publishUnpublishedChallenge($challengeId)
    {
        // Jalankan logika bisnis yang berhubungan dengan 
        // proses mempublikasikan unpublised challenge
        
        // Bila sukses, maka kita tambahkan Challenge ini ke
        // koleksi Published Challenge
        $this->publishedChallengeRepository->add($challenge);
        // Karena challenge ini sudah dipublikasikan, maka kita
        // bangkitkan domain event yang berhubungan dengan kejadian
        // ini
        $challenge->raise(new UnpublishedChallengeWasPublished($challenge->getId()));
        return $challenge;
    
    }
}
```

Tentu saja, domain event dapat dibangkitkan dimana saja selama konteks dari pembangkitan itu tepat secara logika bisnis. Misalnya pasca user baru ditambahkan ke repository, kita dapat membangkitkan event `NewDeveloperWasRegistered`.

## Event Listener

Ketika domain event sudah dipublikasikan, maka seluruh Event Listener akan dijalankan. Dalam contoh use case kita di atas (mempublikasikan Unpublished Challenge), kita memiliki beberapa event listener sebagai berikut:

1. Listener untuk mengirimkan kabar melalui email.
2. Listener untuk menambahkan Challenge ke feed.
3. Listener untuk melakukan tweeting.
4. Listener untuk melakukan logging.

Mari kita buat contoh Event Listener untuk kasus pertama. Contoh dibawah ini menggunakan Laravel 4.2 dan Commander.

```
<?php

class EventMailerServiceProvider extends ServiceProvider
{
    public function register()
    {
        // Ketika seluruh event dari Domain Challenge dibangkitkan, maka kita jalankan ChallengeEventMailer
        // untuk menangani event tersebut.
        \Event::listen('Dicoding.Domain.Challenge.*', 'Dicoding\Infrastructure\Emailing\Challenge\ChallengeEventMailer');
    }
}
// Contoh wujud dari ChallengeEventMailer
class ChallengeEventMailer extends EventListener
{
    public function whenUnpublishedChallengeWasPublished($job, $data)
    {
        // jalankan tugas pengiriman email
    }
    
    public function whenUnpublishedChallengeWasCreated($job, $data)
    {
        
    }
    
    // dsb
}
```

Jadi, ketika `ChallengeEventMailer` menerima event `UnpublishedChallengeWasPublished`, class tersebut akan menjalankan fungsi `whenUnpublishedChallengeWasPublished`. Coba perhatikan bagaimana fungsi tersebut diberi nama sesuai dengan konteks event yang ditanganinya. Memudahkan bukan?

Contoh di atas juga memperlihatkan fungsi tambahan lainnya yang dimiliki oleh `ChallengeEventMailer`. Kita dapat terus menambahkan fungsi-fungsi untuk menangani domain event yang dibangkitkan oleh object dari namespace Challenge.

Bagaimana dengan Event Listener lainnya? Berikut adalah contoh untuk NewsFeed:

```
<?php
class NewsFeedServiceProvider extends ServiceProvider
{
    public function register()
    {
        // Ketika seluruh event dari Domain Challenge dibangkitkan, maka kita jalankan ChallengeFeedListener
        // untuk menangani event tersebut.
        \Event::listen('Dicoding.Domain.Challenge.*', 'Dicoding\Infrastructure\Feeds\Challenges\ChallengeFeedListener');
    }
}
// Contoh wujud dari ChallengeFeedListener
class ChallengeFeedListener extends EventListener
{
    public function whenUnpublishedChallengeWasPublished($job, $data)
    {
        try {
            // tambahkan challenge ke feed
            $challenge = $this->challengeRepository->findById($data);
            $challengeFeed = new ChallengeFeed($challenge);
            $this->feedRepository->add($challengeFeed);
            $job->delete();
            
        } catch (\Exception $e) {
            \Log::error($e);
            throw $e;
        }
        
    }
    
    public function whenUnpublishedChallengeWasCreated($job, $data)
    {
        
    }
    
    // dsb
}
```

Event Listener lainnya akan mengikuti pola Event Listener di atas. Jadi pada dasarnya, kita memiliki Event Listener yang akan mendengarkan Domain Event ketika ia dipublikasikan.

Bila suatu saat kita diberi tugas untuk membuat update status di facebook ketika sebuah unpublished challenge berhasil di publikasikan, kita tinggal membuat Event Listener baru.

### Menjalankan Event Listener di Background

Mengirim email ke pemilik challenge, dan membuat status baru di Twitter, bukanlah proses yang sebentar. Alangkah baiknya bila kita proses-proses yang memakan waktu yang lama ini dijalankan di background. Sehingga ketika proses utama sudah terlaksana, Admin mengetahui hasilnya tanpa harus menunggu tugas-tugas sekunder.

Bagaimana menerapkannya? Kita buat sebuah Abstract Class yang akan menjalankan event di background.

```
<?php
abstract class <?php
abstract class QueuedEventHandler
{
    // Akan dijalankan ketika event diterima
    // Ini adalah mekanisme internal Laravel 4.2
    public function handle($param)
    {
        $this->handleEvent($param);
    }
    
    public function handleEvent($param)
    {
        // Menangkap event
        $currentEventName = Event::firing();
        
        // Memperoleh event name.
        // 
        // Metode ini tidak ditampilkan, karena sangat tergantung
        // dari bagaimana Domain Event disimpan di struktur Aplikasi.
        $domainEvent = $this->getDomainEventName($currentEventName);
        
        // Mengambil namespace dari Event Listener ini.
        // Sehingga kita dapat menjalankannya di sistem Queue
        $currentNamespace = $this->getClientClassFullNamespace();
        
        // Memeriksa apakah Event Listener ini memiliki fungsi untuk
        // menangani event
        if (method_exists($this, "when$domainEvent")) {
            // logging event handling
            Log::info("Event $currentEventName has been captured. And now executing when$domainEvent");
            // Menjalankan handler di background
            // 
            // Ini juga mekanisme internal Laravel 4.2
            Queue::push(
                "$currentNamespace@when$domainEvent",
                [
                    $param
                ]
            );
            
        } else {
            // ketika tidak ada event handler untuk event ini,
            // kita log saja dalam sebuah warning.
            Log::warning("Event $currentEventName has been captured. But no when$domainEvent to handle the event");
        }
    }
}
```

Nah, bila kita ingin menjalankan proses pengiriman email di background, kita tinggal menjadikan `ChallengeEventMailer` meng-extend `QueuedEventHandler` di atas.

```
<?php

// Event handler ini akan berjalan di background
class ChallengeEventMailer extends QueuedEventHandler
{
    public function whenUnpublishedChallengeWasPublished($job, $data)
    {
        try {
            // Kirim email di background
            $job->delete();
            
        } catch (\Exception $e) {
            \Log::error($e);
            throw $e;
        }
        
    }
    
    public function whenUnpublishedChallengeWasCreated($job, $data)
    {
        
    }
    
    // dsb
}
```

### Bagaimana Dengan Testing?

Karena Event Handler di atas dijalankan di background, maka ketika kita menerapkan Integration test pada Use Case di atas, proses event handling belum tentu selesai dijalankan ketika test sudah selesai dieksekusi. Hal ini akan berujung pada laporan yang keliru (false-negative) pada test.

Untuk mengatasi ini, kita dapat menggunakan driver sync ketika aplikasi dijalankan di testing environment. Hal ini mudah dilakukan di Laravel.

Kita tinggal membuat berkas `app/config/testing/queue.php` dan menentukan konfigurasi driver queue di berkas tersebut:

```
<?php
return array(
    'default' => 'sync',
);
```

## Kesimpulan

Kami sudah menjelaskan bagaimana Dicoding memanfaatkan konsep Command dan Domain Event di codebase Dicoding. Contoh-contoh yang kami perlihatkan di dua artikel ini adalah contoh fiktif. Namun, kode yang berjalan di Dicoding saat ini memiliki kemiripan dengan contoh-contoh tersebut.

Dua konsep di atas telah membantu kami dalam mengembangkan Dicoding selama hampir 1.5 tahun perjalanannya. Kami tidak mengatakan bahwa pendekatan ini adalah yang paling sempurna. Namun minimal, alur berfikir yang kami paparkan pada dua artikel ini membantu kami dalam mengembangkan kode yang lebih mudah untuk dikelola.

Semoga dua artikel ini ada manfaatnya. Dan bila teman-teman memiliki pendekatan yang dirasa sangat membantu teman-teman dalam menulis kode yang mudah dikelola, saya sangat berharap teman-teman mau membaginya.

Mari kita belajar bersama.