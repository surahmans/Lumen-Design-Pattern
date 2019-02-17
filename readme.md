

# Lumen Design Pattern

> Tulisan ini dibuat sebagai referensi dasar dalam membangun sebuah aplikasi berbasis Lumen framework. Adapun design pattern yang akan digunakan dalam guide ini adalah repository design pattern. Sedangkan design pattern yang lainnya hanya sebagai pelengkap saja.

Adapun beberapa aturan yang bisa gunakan adalah:

1. ## Repository Pattern

   Karena fokus utama kita adalah repository pattern. Maka sangatlah tepat menjadikan ini sebagai pembuka awal aturan. Repository pattern bertujuan untuk memisahkan business logic dengan database. Untuk itu persistence layer harus berada dalam repository. Sebagai bahan referensi, silahkan merujuk pada artikel dari [Bosnadev](https://bosnadev.com/2015/03/07/using-repository-pattern-in-laravel-5/) ataupun package dari [Prettus](https://github.com/andersao/l5-repository).

2. ## Fractal

   Untuk urusan presentation dan transformation data, kita akan menggunakan [Fractal](https://github.com/andersao/l5-repository). Anda yang sudah sering membuat RESTful APIs tentu tahu akan yang satu ini. 

3. ## Perhatikan Controllermu!

   Dengan dua hal di atas sudah mampu mempercantik controller kita. Karena kita tidak lagi mencampur-adukan bussiness logic yang biasanya ada di controller dengan data. Namun, ada satu hal lagi yang perlu diperhatikan agar controller Anda makin indah. Yakni Anda harus mampu membuat satu buah method yang dapat menerima berbagai macam serangan tipe data. Seperti `array`, `model`, `collection`, `null` maupun instance dari `Illuminate\Pagination\LengthAwarePaginator`.

   ```php
   //array
   $this->responseSuccess([
   	'message' => 'You are rock!'
   ]);
   
   //model
   $this->responseSuccess($this->repository->find($id = 1));
   
   //collection
   $this->responseSuccess($this->repository->getAll());
   
   //null
   $this->responseSuccess(null);
   
   //paginator
   $this->responseSuccess($this->repository->paginate($page = 1));
   
   ```

   Tentu saja dibalik method `responseSuccess` itu terdapat banyak peran yang dimainkan oleh [Transformer](https://fractal.thephpleague.com/transformers/) dan [Pagination](https://fractal.thephpleague.com/pagination/) dari Fractal.

4. ## Form Request Untuk Validasi

   Biasanya controller memang dijadikan alasan untuk melakukan berbagai macam business logic. Tapi kali ini kita dapat memanfaatkan [Form Request Validation](https://laravel.com/docs/5.7/validation#form-request-validation) untuk membersihkan controller kita dari hal tersebut. Sayangnya, form request validation ini tidak tersedia di Lumen. Untuk itu kita mengakalinya dengan cara menginstall package yang bisa temukan di packagist.org. Jika tidak mau repot, Anda dapat menggunakan karya dari [Pearl](https://packagist.org/packages/pearl/lumen-request-validate) untuk hal ini.

5. ## Observer Pattern

   Tidak bisa dipungkuri ketika kita meletakan business logic ke dalam controller, cepat atau lambat controller kita akan menjadi besar (Dalam makna yang negatif). Karena itu, kita dapat memanfaatkan observer pattern yang dimiliki Laravel Lumen ini. Observer adalah the next option ketika kita membutuhkan banyak event dan listener pada sebuah model. 

   Sebagai contoh kita memiliki sebuah model `User` dan `Article`. Yang mana pada user tersebut terdapat tiga role yakni `super admin`, `admin`, `normal user`. Masing-masing role tersebut membutuhkan perlakuan khusus ketika membuat artikel. Maka ketika kita menggunakan observer kode kita akan mungkin terlihat seperti ini.

   ```php
   class UserObserver
   {
       public function creating($user)
       {
           $loginUser = Auth::user();
           
           if ($loginUser->role == 'super-admin') {
               event(new SuperAdminCreatingArticle($user));
           }
           
            if ($loginUser->role == 'admin') {
               event(new AdminCreatingArticle($user));
           }
           
            if ($loginUser->role == 'normal-user') {
               event(new NormalUserCreatingArticle($user));
           }
       }
   }
   ```

   Selanjutkan kita tinggal membuat listener untuk melakukan tugasnya masing-masing. Contoh penggunaan observer dapat dilihat pada [artikel medium Pete Houston](https://medium.com/@petehouston/observe-things-in-laravel-d43be82f4ba4) dan event apa saja yang tersedia terdapat di [dokumentasi resmi Laravel.](https://laravel.com/docs/5.7/eloquent#events) 

6. ## Response Success

   Mari buat standarisasi untuk response kita. Pada response harus terdapat status code, message dan juga data itu sendiri. Adapun struktur output yang kita harapkan adalah seperti ini.

   ```json
   {
       "status": {
           "code": 0,
           "message": "success"
       },
       "data": {
           "name": "PUBG"
       }
   }
   ```

   Pada Fractal, hal tersebut tidak bisa kita lakukan tanpa modifikasi. Karena secara otomatis ia akan membuat struktur seperti ini.

   ```php
   // Item
   [
       'data' => [
           'foo' => 'bar'
       ],
   ];
   
   // Collection
   [
       'data' => [
           [
               'foo' => 'bar'
           ]
       ],
   ];
   ```

   Ya! Fractal selalu menyematkan "data" pada responsenya. Untuk itu kita dapat memanfaatkan fitur  [serializers](https://fractal.thephpleague.com/serializers/) yang telah disediakan Fractal. Sehingga kita dapat melempar variabel `$data` ke dalam method yang menangani success response di atas.

   ```php
    return response()->json([
        'status' => [
            'code'    => 0,
            'message' => $message
        ],
        'data' => $data
    ], $httpStatusCode);
   ```

   Sehingga berbagai macam bentuk response akan kita dapati seperti ini.

   ```json
   //Item response dengan collection platforms di dalamnya
   {
       "status": {
           "code": 0,
           "message": "success"
       },
       "data": {
           "id": 1,
           "name": "PUBG",
           "platforms": {
               "data": [
                   {
                       "id": 2,
                       "name": "PC"
                   },
                   {
                       "id": 1,
                       "name": "Mobile"
                   }
               ]
           }
       }
   }
   ```
   ```json
   //Item response dengan item address di dalamnya
   {
       "status": {
           "code": 0,
           "message": "success"
       },
       "data": {
           "id": 1,
           "name": "John Doe",
           "address": {
               "city": "West Judge",
   			"streetName": "Keegan Trail",                   
              	"streetAddress": "439 Karley Loaf Suite 897",
   			"postcode": 17916
           }
       }
   }
   ```

   ```json
   //Collection response
   {
       "status": {
           "code": 0,
           "message": "success"
       },
       "data": {
           "data": [
               {
                   "id": 1,
                   "name": "PUBG"
               },
               {
                   "id": 2,
                   "name": "Fornite"
               },
               {
                   "id": 4,
                   "name": "DOTA 2"
               },
               {
                   "id": 5,
                   "name": "CS GO"
               }
           ]
       }
   }
   ```

   ```json
   //Paginator response
   {
       "status": {
           "code": 0,
           "message": "success"
       },
       "data": {
           "data": [
               {
                   "id": 1,
                   "name": "PUBG"
               },
               {
                   "id": 2,
                   "name": "Fornite"
               }
           ],
           "meta": {
               "pagination": {
                   "total": 5,
                   "count": 2,
                   "per_page": 2,
                   "current_page": 1,
                   "total_pages": 3,
                   "links": {
                       "next": "http://localhost/v1/tournaments?page=2"
                   }
               }
           }
       }
   }
   ```

   

7. ## Response Fail

   Terakhir adalah response ketika terjadi kesalahan. Jika pada success kita menggunakan code `0`, maka pada fail kita menggunakan error code `1`. Penerapan response fail ini terbagi menjadi dua tempat.

   1. ### Form Request Validation

      Pada form request validation kita menempatkan error code sebagai tanda bahwa data tidak berhasil sesuai validasi. Kita dapat meng-extend abstract class `Pearl\RequestValidate\RequestAbstract` (Dari package Form Request Validation di atas) dan membuat kode berikut untuk membuat response sesuai struktur yang sudah kita sepakati sebelumnya.

      ```php
      protected function formatErrors(Validator $validator)
      {
          return new JsonResponse([
              'status' => [
                  'code'    => 1,
                  'message' => $validator->getMessageBag()->first()
              ]
          ], 422);
      }
      ```

   1. ### Exception

      Pada exception kita perlu membuat sebuah class `BaseException` yang memiliki method untuk memberikan response fail sesuai struktur di atas. Contohnya adalah seperti berikut.

      ```php
       public function jsonErrorResponse()
       {
           return response()->json([
               'status' => [
                   'code'    => $this->error_code,
                   'message' => $this->message
               ]
           ], $this->http_status_code);
       }
      ```

      Maka ketika terjadi sebuah kesalahan pada kode program, kita tinggal melempar exception seperti contoh berikut.

      ```php
      try {
          //business logic
      } catch (\Exception $e) {
          throw new \App\Exceptions\BaseException('Someting went wrong!');
      }
      ```

      Dan pastikan Anda juga memodifikasi `App\Exceptions\Handler` class agar bisa merender method `jsonErrorResponse` kita di atas.

      ```php
      public function render($request, Exception $exception)
      {
          if ($exception instanceof \App\Exceptions\BaseException) {
              return $exception->jsonErrorResponse();
          }
      
          return parent::render($request, $exception);
      }
      ```

      

### Conclusion

Dengan menerapkan semua hal di atas, maka seharusnya controller Anda setiap methodnya tidak akan lebih dari 3 baris kode!

```php
 public function store(StoreArticleRequest $request)
 {
     $article = $this->repo->create($request->all());

     return $this->responseSuccess($article);
 }
```

Karena kita telah memisahkan berbagai macam hal seperti berikut:

1. Data, pada repository
2. Business logic, pada listener
3. Validation, pada form request




