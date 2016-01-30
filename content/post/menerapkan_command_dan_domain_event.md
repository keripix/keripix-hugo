+++
date = "2016-01-27T14:43:52+07:00"
description = "Command dan Domain Events adalah dua konsep yang benar-benar telah membantu pengembangan Dicoding hingga saat ini."
title = "Menerapkan Command dan Domain Event"

+++

![Evaluasi perjalanan pengembangan Dicoding](https://cdn-images-1.medium.com/max/2000/1*LIYtUrLIm1oBr1537VgYPA.jpeg)

Pengembangan Dicoding sudah berjalan lebih dari setahun. Selama perjalanan ini, ada dua konsep yang sampai sekarang masih bertahan, yaitu Command dan Domain Events. Dua konsep tersebut, benar-benar telah membantu pengembangan Dicoding hingga saat ini.

Di artikel ini, kami akan mencoba menjelaskan bagaimana Command dan Domain Events diterapkan di codebase Dicoding.

## Command

### Apa itu Command

Command adalah sebuah perintah. Ketika ia diimplementasikan, perintah tersebut berhubungan dengan sebuah use case pada Aplikasi.

Contoh command adalah `CreateUnpublishedChallenge`, `ChangeMemberPassword`, `UpdateMemberProfile`, dsb.

Bila kita melihat contoh command di atas, kita kurang lebih dapat menebak dampak pasca command dijalankan (eksekusi).

Bila command `CreateUnpublishedChallenge` dieksekusi, maka sebuah `Challenge` yang belum terpublikasikan akan terbuat.

Penamaan command seperti contoh di atas adalah hal yang disengaja. Tidak hanya karena command-command tersebut memiliki hubungan eksplisit dengan hasil pasca ia dijalankan. Penamaan tersebu juga berhubungan erat dengan bahasa bisnis yang digunakan diluar pengembangan.

Coba kita bandingkan dengan bahasa `Controller` yang bisa jadi menandakan proses pembuatan Challenge:

```
<?php
class ChallengeController
{
    /**
     * POST action 
     */
    public function store() {
        
    }
}

class ChallengeController
{
    /**
     * POST action 
     */
    public function createAction() {
        
    }
}
```

Kita mungkin bisa menebak bahwa kedua fungsi di atas (`store` & `createAction`), sama sama membuat sebuah resource (dalam hal ini `Challenge`). Namun, menurut kami `CreateUnpublishedChallenge` lebih tegas menandakan apa yang sedang dibuat.

Penamaan yang secara eksplisit menandakan tanggung jawab dari sebuah class ataupun fungsi, dapat memudahkan pengelolaan kode. Karena kita butuh waktu yang lebih sedikit untuk mencari tahu apa tujuan dari kode tersebut.

Sebuah Command juga menyimpan seluruh data yang diperlukan agar command tersebut dapat berjalan. Misalnya:

```
<?php

class CreateUnpublishedChallenge
{
    public function __construct($name, $summary, $description, $winningPoint, $winningQuota)
    {
        // constructing
    }
}
```

Kita dapat melihat, bahwa untuk membuat sebuah *Unpublished Challenge*, sistem membutuhkan `name`, `summary`, `description`, `winningPoint`, dan `winningQuota`.

### Bagaimana Command digunakan?

*Command* dijalankan dengan cara menyerahkannya ke *Command Bus*. *Command Bus* bertugas menemukan object mana yang akan menangani (*handle*) command di atas (Command Handler: object yang menjalankan Command).

Di Dicoding, kami menggunakan sebuah package yang bernama [Commander](https://github.com/laracasts/Commander) untuk menangani eksekusi Command. Dengan memanfaatkan Commander, kita dapat mengeksekusi sebuah Command dengan cara berikut:

```
<?php

class ChallengeController
{
    public function store()
    {
        try {
            // Mengeksekusi Command
            $this->execute(CreateUnpublishedChallenge::class);
            
        } catch (Exception $e) {
            // ...
        }
    }
}
```

Tentu, penanganan command tidak harus seperti cara di atas. Bila kita menggunakan [SimpleBus](http://simplebus.github.io/MessageBus/doc/command_bus.html), maka bentuknya adalah seperti contoh berikut

```
<?php

class ChallengeController
{
    public function store()
    {
        try {
            $this->commandBus()->handle(new CreateUnpublishedChallenge(
                'Nama',
                'Ringkasan',
                'Deskripsi',
                1000,
                10
            ));
            
        } catch (Exception $e) {
            // ...
        }
    }
}
```

Namun, garis besarnya adalah kita tinggal mengeksekusi Command, dan Command Bus akan menemukan handlernya.

### Siapa yang mengeksekusi Command

Pada beberapa contoh di atas, command dieksekusi oleh `Controller`. Namun sebenarnya, kita dapat mengeksekusi Command di mana saja. Misalnya, command berikut akan dijalankan oleh cron job untuk memberi kabar kepada para Admin mengenai status pending task di Dicoding:

```
<?php

use Dicoding\ApplicationService\Administration\NotifyPendingTaskCommand;
use Illuminate\Console\Command;

class DailyTaskNotifier extends Command
{
    // code lain disembunyikan agar contoh ini menjadi lebih ringkas
    
    public function fire()
    {
        try {
            $commandBus = App::make('Laracasts\Commander\CommandBus');
            
            $commandBus->execute(new NotifyPendingTaskCommand());
            
        } catch (Exception $ex) {
            Log::error($ex);
        }
    }
}
```

### Command Handler

Tiap Command, memiliki satu buah Command Handler yang bertanggung jawab untuk menjalankan Command tersebut. Di Command Handler inilah kita menuliskan implementasi dari sebuah use case.

```
<?php

use Dicoding\ApplicationService\Challenges;

class CreateUnpublishedChallengeHandler
{
    public function handle(CreateUnpublishedChallenge $command)
    {
        try {
            
            // jalankan use case       
            
        } catch (\Exception $e) {
            
        }
    }
}
```

Bagaimana Command Handler di atas dijalankan? Ini adalah tanggung jawab dari Command Bus. Command Bus bertanggung jawab untuk mencari tahu, Command Handler mana yang akan menjalankan sebuah Command.

### Keuntungan Menggunakan Command

Dari beberapa contoh di atas, kita dapat melihat bahwa logika bisnis tidak lagi berada di class yang mengeksekusi command. Ada beberapa alasan mengapa pemisahan ini memudahan kita dalam mengelola kode:

#### Memisahkan kode bisnis dengan mekanisme delivery.

Yang dimaksud dengan mekanisme delivery disini adalah bagaimana aplikasi dijalankan. Contohnya adalah web framework. Mekanisme delivery memiliki bahasa yang berbeda dengan bahasa bisnis.

Contohnya `ChallengeController::store` adalah bahasa untuk Controller ketika menangani POST Request. Sementara CreateUnpublishedChallenge adalah bahasa bisnis untuk menangani permintaan membuat sebuah Unpublished Challenge.

Kita bisa menjalankan sebuah Command di sebuah Controller. Kita juga bisa menjalankan sebuah Command dari Command Line. Object yang menerapkan kode bisnis, tidak perlu tahu bagaimana ia dijalankan. Bagaimana kode bisnis dijalankan adalah tanggung jawab dari mekanisme delivery.

#### Mengetahui use case apa saja yang sudah diimplementasikan

Kita dapat mengetahui Use Case apa saja yang diterapkan oleh aplikasi kita, dengan cara melihat Command Objects. Dan menurut Uncle Bob, [aplikasi kita perlu menegaskan fungsinya](https://www.youtube.com/watch?v=WpkDN78P884). Dan fungsi ini dapat kita lihat di Command Objects.

#### Memudahkan Mencari Object Yang Mengimplementasikan Sebuah Use Case

Keuntungan berikutnya adalah kita dapat dengan lebih mudah mencari kode object yang mengimplementasikan pembuatan *Unpublished Challenge* (kita akan membahas soal Command Handler).

Hal di atas lebih masuk akal misalnya, bila kita punya dua logika pembuatan Challenge yang berbeda tergantung siapa yang membuatnya. Misalnya, bila yang membuat adalah Admin, maka Challenge yang terbuat langsung dipublikasikan. Dan bila yang membuat adalah non-Admin, maka Challenge yang dibuat tidak lansung dipublikasikan.

Untuk kasus pertama, maka kita tinggal menjalankan `CreatePublishedChallenge`. Dan untuk kasus kedua, kita menjalankan `CreateUnpublishedChallenge`. Untuk mencari tahu implementasi dari kedua use case di atas, kita tinggal membuka Command Handler untuk tiap Command.

#### Don’t Repeat Yourself

Ketika kita mengimplementasi sebuah Use Case di satu tempat, kita telah menerapkan prinsip DRY (Don’t Repeat Yourself). Prinsip DRY mendorong kita untuk menuliskan implementasi dari suatu pengetahuan di satu tempat yang spesifik.

Bila kita memiliki lebih dari satu object yang sama-sama menerapkan implementasi pembuatan Unpublished Challenge, misalnya, maka bila terjadi perbedaan antara tiap implementasi di atas, object manakah yang implementasinya lebih tepat?

Kejadian di atas menandakan adanya ambiguitas pengetahuan. Kejadian tersebut akan menjadi lebih kentara ketika sistem bertambah besar dan kebutuhan bisnis terus berkembang.

Situasi akan menjadi berbeda, bila kita memiliki satu otoritas yang tahu bagaimana mengimplementasikan pembuatan Unpublished Challenge. Kita tahu object mana yang memiliki pengetahuan terpercaya dalam menerapkan Use Case pembuatan Unpublished Challenge.

Kondisi di atas memberikan kita dua keuntungan.

**Yang pertama**, kita dapat menggunakan implementasi sebuah Use Case di lebih dari satu tempat yang berbeda.

Misalnya, logika registrasi member dapat digunakan lebih dari sekali, di dua skenario yang berbeda. Contohnya ketika mendaftar secara manual ataupun memanfaatkan Social Account. Dua skenario ini bisa jadi diterapkan di dua Controller yang berbeda. Namun kedua Controller ini sama-sama menjalankan `RegisterDeveloper` command.

```
<?php

class EmailMembershipController
{
    public function store()
    {
        try {
            $this->execute(RegisterDeveloper::class);
            
        } catch (Exception $e) {
            // ...
        }
    }
}

class FacebookMembershipController
{
    public function store()
    {
        try {
            $this->execute(RegisterDeveloper::class);
            
        } catch (Exception $e) {
            // ...
        }
    }
}
```

**Keuntungan kedua** ketika menerapkan DRY adalah bila kita perlu mengubah suatu logika tertentu, misalnya logika tentang siapa yang bisa memenangkan sebuah Challenge, kita hanya melakukan perubahan di satu tempat. Mengapa?

Karena hanya satu tempat itulah yang memiliki pengetahuan untuk menentukan siapa yang dapat memenangkan sebuah Challenge.

## Domain Events

Konsep kedua yang memiliki peran positif di pengembangan Dicoding adalah Domain Events. Namun pembahasan ini akan kami lakukan di artikel yang terpisah.

Semoga artikel ini ada manfaatnya. Mohon masukan teman-teman bila ada konsep yang keliru, ada masukan mengenai penerapan yang lebih efektif, atau ada penjelasan yang kurang jelas.