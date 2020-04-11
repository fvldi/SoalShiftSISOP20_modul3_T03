# Soal Shift SISOP20 Modul 3
## Kelompok T03
### 1. Muhamad Rifaldi - 05311840000022
### 2. Dimas Pramudya H - 05311840000037


### Soal Nomer 3
```
Buatlah sebuah program dari C untuk mengkategorikan file. Program ini akan
memindahkan file sesuai ekstensinya (tidak case sensitive. JPG dan jpg adalah
sama) ke dalam folder sesuai ekstensinya yang folder hasilnya terdapat di working
directory ketika program kategori tersebut dijalankan.
  ● Semisal program dijalankan:
  
  # File kategori terletak di /home/izone/kategori
  $ ./kategori -f path/to/file1.jpg path/to/file2.c path/to/file3.zip
    
  #Hasilnya adalah sebagai berikut
  /home/izone
  |-jpg
  |--file1.jpg
  |-c
  |--file2.c
  |-zip
  |--file3.zi
    
  ● Pada opsi -f tersebut, user bisa menambahkan argumen file yang bisa
    dikategorikan sebanyak yang user inginkan seperti contoh di atas.
  ● Pada program kategori tersebut, folder jpg,c,zip tidak dibuat secara manual,
    melainkan melalui program c. Semisal ada file yang tidak memiliki ekstensi,
    maka dia akan disimpan dalam folder “Unknown”.
  ● Program kategori ini juga menerima perintah (*) seperti di bawah;
  
  $ ./kategori \*
    
  ● Artinya mengkategori seluruh file yang ada di working directory ketika
    menjalankan program C tersebut.
  ● Selain hal itu program C ini juga menerima opsi -d untuk melakukan kategori
    pada suatu directory. Untuk opsi -d ini, user hanya bisa menginput 1 directory
    saja, tidak seperti file yang bebas menginput file sebanyak mungkin.

  $ ./kategori -d /path/to/directory/
  
  ● Hasilnya perintah di atas adalah mengkategorikan file di /path/to/directory dan
    hasilnya akan disimpan di working directory di mana program C tersebut
    berjalan (hasil kategori filenya bukan di /path/to/directory).
  ● Program ini tidak rekursif. Semisal di directory yang mau dikategorikan, atau
    menggunakan (*) terdapat folder yang berisi file, maka file dalam folder
    tersebut tidak dihiraukan, cukup file pada 1 level saja.
  ● Setiap 1 file yang dikategorikan dioperasikan oleh 1 thread agar bisa berjalan
    secara paralel sehingga proses kategori bisa berjalan lebih cepat. Dilarang
    juga menggunakan fork-exec dan system.
  ● Silahkan download soal3.zip sebagai percobaan. Namun silahkan
    dicoba-coba sendiri untuk kemungkinan test case lainnya yang mungkin
    belum ada di soal3.zip.
```
### Source Code : 
```c
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <unistd.h>
#include <wait.h>
#include <time.h>
#include <stdbool.h>
#include <dirent.h>
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <pthread.h>
#include <ctype.h>
#include <libgen.h>

#define TRUE 1
#define FALSE 0
#define F 2
#define D 3
#define STAR 4

int isDirectory(const char *path) {
   struct stat statbuf;
   if (stat(path, &statbuf) != 0)
       return FALSE;
   return S_ISDIR(statbuf.st_mode);
}

int checkAllFileExist(int argc, char *argv[]) {
    int i;
    char curr_dirr[1000];
    getcwd(curr_dirr, sizeof(curr_dirr));
    strcat(curr_dirr, "/");
    for ( i = 2; i < argc; i++)
    {
        char temp[1000];
        if(argv[i][0] != '/') {
            strcpy(temp, curr_dirr);
            strcat(temp, argv[i]);
        }else strcpy(temp, argv[i]);
        struct stat buffer;
        if(stat(argv[i], &buffer) != 0 || isDirectory(temp)) return FALSE;
    }
    return TRUE;
}

const char *get_filename_ext(const char *filename) {
    const char *dot = strrchr(filename, '.');
    if(!dot || dot == filename) return "";

    return dot;
}

int checkMainArg(int argc, char *argv[]) {
    if(argc < 2)
        return FALSE;
    if(argv[1][0] == '*' && strlen(argv[1]) == 1 && argc == 2)
        return STAR;   
    if(argv[1][0] == '-' && strlen(argv[1]) == 2)
        switch (argv[1][1])
        {
        case 'f':
            return (argc < 3)? FALSE : (strlen(argv[1]) != 2)? FALSE : (!checkAllFileExist(argc, argv))? FALSE : F;
        case 'd':
            return (argc != 3)? FALSE : (strlen(argv[1]) != 2)? FALSE : (!isDirectory(argv[2]))? FALSE : D;
        }

    return FALSE;
}

void moveFileToNewPath(char *source, char *dest) {
    FILE *fp1, *fp2;
    int ch;
    
    fp1 = fopen(source, "r");
    fp2 = fopen(dest, "w");

    while ((ch = fgetc(fp1)) != EOF) fputc(ch, fp2);
    
    fclose(fp1);
    fclose(fp2);

    remove(source);
}

void *moveFile( void *ptr )
{
    char *param = (char *)ptr;
    const char *extension_temp = get_filename_ext(param);
    int ch, i, length = strlen(extension_temp);
    char extension[100];
    char *filename = basename(param);
    char new_path[1000],temp[1000];

    memset(new_path, 0, sizeof(new_path));
    memset(extension, 0, sizeof(extension));

    for ( i = 1; i < length; i++)
    {
        extension[i - 1] = tolower(extension_temp[i]);
    }
    
    
    if(!strlen(extension_temp)) {
        if(!isDirectory("Unknown"))mkdir("Unknown", 0777);
        strcpy(new_path, "Unknown/");
        strcat(new_path, filename);
    }else{
        if(!isDirectory(extension))mkdir(extension, 0777);
        strcpy(new_path, extension);
        strcat(new_path, "/");
        strcat(new_path, filename);
    }

    moveFileToNewPath(param, new_path);
}

int main(int argc, char *argv[]) {
    int check = checkMainArg(argc, argv), i = 0, iret[10000], let = 1;

    if(!check) {
        printf("Argumen Salah!\n");
        exit(EXIT_FAILURE);
    }

    char curr_dirr[1000];
    pthread_t threads[10000];
    struct dirent *ep;     
    DIR *dp;

    switch (check)
    {
    case F:
        for (i = 2; i < argc; i++)
        {
            char *arr = argv[i];
            iret[i - 2] = pthread_create(&threads[i - 2], NULL, moveFile, (void*) arr);
            
            if(iret[i - 2])
            {
                fprintf(stderr,"Error - pthread_create() return code: %d\n",iret[i - 2]);
                exit(EXIT_FAILURE);
            }

            pthread_join(threads[i - 2], NULL); 
        }
        break;
    case D:
        getcwd(curr_dirr, sizeof(curr_dirr));
        if(argv[2][0] != '/'){
            strcat(curr_dirr, "/");
            strcat(curr_dirr, argv[2]);
            if(argv[2][strlen(argv[2]) - 1] != '/')
                strcat(curr_dirr, "/");
        }
        dp = opendir (argv[2]);
        if(dp != NULL) {
            while (ep = readdir (dp)){
                char temp[1000];
                strcpy(temp, curr_dirr);
                strcat(temp, ep->d_name);
                if(strcmp(".",ep->d_name) !=0 && strcmp("..",ep->d_name) != 0 && !isDirectory(temp)) {
                    strcpy(temp, argv[2]);
                    if(argv[2][strlen(argv[2]) - 1] != '/') strcat(temp, "/");
                    strcat(temp, ep->d_name);
                    char *arr = temp;
                    iret[i] = pthread_create(&threads[i], NULL, moveFile, (void*) arr);
            
                    if(iret[i])
                    {
                        fprintf(stderr,"Error - pthread_create() return code: %d\n",iret[i]);
                        exit(EXIT_FAILURE);
                    }

                    pthread_join(threads[i], NULL); 
                }
                i++;
            }
        }
        break;
    case STAR:
        if(getcwd(curr_dirr, sizeof(curr_dirr)) != NULL) {
            dp = opendir (curr_dirr);
            if(dp != NULL) {
                while (ep = readdir (dp)){
                    char temp[1000];
                    strcpy(temp, curr_dirr);
                    strcat(temp, "/");
                    strcat(temp, ep->d_name);
                    if(strcmp(".",ep->d_name) !=0 && strcmp("..",ep->d_name) != 0 && !isDirectory(temp)) {
                        char *arr = temp;
                        iret[i] = pthread_create(&threads[i], NULL, moveFile, (void*) arr);
                
                        if(iret[i])
                        {
                            fprintf(stderr,"Error - pthread_create() return code: %d\n",iret[i]);
                            exit(EXIT_FAILURE);
                        }

                        pthread_join(threads[i], NULL); 
                    }
                    i++;
                }
            }
        }
        break;
    }
}
```

## Penjelasan : 

- `isDirectory` cek apakah path yang dimasukkan adalah path ke directory atau file
```c
int isDirectory(const char *path) {
   struct stat statbuf;
   if (stat(path, &statbuf) != 0)
       return FALSE;
   return S_ISDIR(statbuf.st_mode);
}
```

- `checkAllFileExist` cek apakah semua path file yang dimasukkan ada
```c
int checkAllFileExist(int argc, char *argv[]) {
    int i;
    char curr_dirr[1000];
    getcwd(curr_dirr, sizeof(curr_dirr));
    strcat(curr_dirr, "/");
    for ( i = 2; i < argc; i++)
    {
        char temp[1000];
        if(argv[i][0] != '/') {
            strcpy(temp, curr_dirr);
            strcat(temp, argv[i]);
        }else strcpy(temp, argv[i]);
        struct stat buffer;
        if(stat(argv[i], &buffer) != 0 || isDirectory(temp)) return FALSE;
    }
    return TRUE;
}
```

- `get_filename_ext` untuk mendapat ekstensi dari suatu file
```c
const char *get_filename_ext(const char *filename) {
    const char *dot = strrchr(filename, '.');
    if(!dot || dot == filename) return "";

    return dot;
}
```

- `checkMainArg` checking argumen yang dimasukkan ke program
```c
int checkMainArg(int argc, char *argv[]) {
    if(argc < 2)
        return FALSE;
    if(argv[1][0] == '*' && strlen(argv[1]) == 1 && argc == 2)
        return STAR;   
    if(argv[1][0] == '-' && strlen(argv[1]) == 2)
        switch (argv[1][1])
        {
        case 'f':
            return (argc < 3)? FALSE : (strlen(argv[1]) != 2)? FALSE : (!checkAllFileExist(argc, argv))? FALSE : F;
        case 'd':
            return (argc != 3)? FALSE : (strlen(argv[1]) != 2)? FALSE : (!isDirectory(argv[2]))? FALSE : D;
        }

    return FALSE;
}
```

- `moveFileToNewPath` memindahkan file ke path baru
```c
void moveFileToNewPath(char *source, char *dest) {
    FILE *fp1, *fp2;
    int ch;
    
    fp1 = fopen(source, "r");
    fp2 = fopen(dest, "w");

    while ((ch = fgetc(fp1)) != EOF) fputc(ch, fp2);
    
    fclose(fp1);
    fclose(fp2);

    remove(source);
}
```

- `moveFile` fungsi untuk memindahkan ke suatu directory
```c
void *moveFile( void *ptr )
{
    char *param = (char *)ptr;
    const char *extension_temp = get_filename_ext(param);
    int ch, i, length = strlen(extension_temp);
    char extension[100];
    char *filename = basename(param);
    char new_path[1000],temp[1000];

    memset(new_path, 0, sizeof(new_path));
    memset(extension, 0, sizeof(extension));

    for ( i = 1; i < length; i++)
    {
        extension[i - 1] = tolower(extension_temp[i]);
    }
    
    
    if(!strlen(extension_temp)) {
        if(!isDirectory("Unknown"))mkdir("Unknown", 0777);
        strcpy(new_path, "Unknown/");
        strcat(new_path, filename);
    }else{
        if(!isDirectory(extension))mkdir(extension, 0777);
        strcpy(new_path, extension);
        strcat(new_path, "/");
        strcat(new_path, filename);
    }

    moveFileToNewPath(param, new_path);
}
```

- Lalu di main function 
```c
int main(int argc, char *argv[]) {
    int check = checkMainArg(argc, argv), i = 0, iret[10000], let = 1;

    if(!check) {
        printf("Argumen Salah!\n");
        exit(EXIT_FAILURE);
    }

    char curr_dirr[1000];
    pthread_t threads[10000];
    struct dirent *ep;     
    DIR *dp;

    switch (check)
    {
    case F:
        for (i = 2; i < argc; i++)
        {
            char *arr = argv[i];
            iret[i - 2] = pthread_create(&threads[i - 2], NULL, moveFile, (void*) arr);
            
            if(iret[i - 2])
            {
                fprintf(stderr,"Error - pthread_create() return code: %d\n",iret[i - 2]);
                exit(EXIT_FAILURE);
            }

            pthread_join(threads[i - 2], NULL); 
        }
        break;
    case D:
        getcwd(curr_dirr, sizeof(curr_dirr));
        if(argv[2][0] != '/'){
            strcat(curr_dirr, "/");
            strcat(curr_dirr, argv[2]);
            if(argv[2][strlen(argv[2]) - 1] != '/')
                strcat(curr_dirr, "/");
        }
        dp = opendir (argv[2]);
        if(dp != NULL) {
            while (ep = readdir (dp)){
                char temp[1000];
                strcpy(temp, curr_dirr);
                strcat(temp, ep->d_name);
                if(strcmp(".",ep->d_name) !=0 && strcmp("..",ep->d_name) != 0 && !isDirectory(temp)) {
                    strcpy(temp, argv[2]);
                    if(argv[2][strlen(argv[2]) - 1] != '/') strcat(temp, "/");
                    strcat(temp, ep->d_name);
                    char *arr = temp;
                    iret[i] = pthread_create(&threads[i], NULL, moveFile, (void*) arr);
            
                    if(iret[i])
                    {
                        fprintf(stderr,"Error - pthread_create() return code: %d\n",iret[i]);
                        exit(EXIT_FAILURE);
                    }

                    pthread_join(threads[i], NULL); 
                }
                i++;
            }
        }
        break;
    case STAR:
        if(getcwd(curr_dirr, sizeof(curr_dirr)) != NULL) {
            dp = opendir (curr_dirr);
            if(dp != NULL) {
                while (ep = readdir (dp)){
                    char temp[1000];
                    strcpy(temp, curr_dirr);
                    strcat(temp, "/");
                    strcat(temp, ep->d_name);
                    if(strcmp(".",ep->d_name) !=0 && strcmp("..",ep->d_name) != 0 && !isDirectory(temp)) {
                        char *arr = temp;
                        iret[i] = pthread_create(&threads[i], NULL, moveFile, (void*) arr);
                
                        if(iret[i])
                        {
                            fprintf(stderr,"Error - pthread_create() return code: %d\n",iret[i]);
                            exit(EXIT_FAILURE);
                        }

                        pthread_join(threads[i], NULL); 
                    }
                    i++;
                }
            }
        }
        break;
    }
}
```


## Soal Nomer 4
```
Norland adalah seorang penjelajah terkenal. Pada suatu malam Norland menyusuri
jalan setapak menuju ke sebuah gua dan mendapati tiga pilar yang pada setiap
pilarnya ada sebuah batu berkilau yang tertancap. Batu itu berkilau di kegelapan dan
setiap batunya memiliki warna yang berbeda.

Norland mendapati ada sebuah teka-teki yang tertulis di setiap pilar. Untuk dapat
mengambil batu mulia di suatu pilar, Ia harus memecahkan teka-teki yang ada di
pilar tersebut. Norland menghampiri setiap pilar secara bergantian.

- Batu mulia pertama. Emerald. Batu mulia yang berwarna hijau mengkilat. Pada
  batu itu Ia menemukan sebuah kalimat petunjuk. Ada sebuah teka-teki yang berisi:
    1. Buatlah program C dengan nama "​ 4a.c​ ", yang berisi program untuk
       melakukan perkalian matriks. Ukuran matriks pertama adalah ​ 4x2​ , dan
       matriks kedua ​ 2x5​ . Isi dari matriks didefinisikan ​ di dalam kodingan. Matriks
       nantinya akan berisi angka 1-20 (​ tidak perlu​ dibuat filter angka).
    2. Tampilkan matriks hasil perkalian tadi ke layar.
    
- Batu kedua adalah Amethyst. Batu mulia berwarna ungu mengkilat. Teka-tekinya
adalah:
    1. Buatlah program C kedua dengan nama "​ 4b.c​ ". Program ini akan
       mengambil variabel ​ hasil perkalian matriks dari program "4a.c" (program
       sebelumnya), dan tampilkan hasil matriks tersebut ke layar.
    (​ Catatan!​ : gunakan shared memory)
    2. Setelah ditampilkan, berikutnya untuk setiap angka dari matriks
       tersebut, carilah nilai ​ faktorialnya​ , dan tampilkan hasilnya ke layar dengan
       format seperti matriks.
          Contoh: misal array [[1, 2, 3, 4], [5, 6, 7, 8], [9, 10, 11, 12], ...],
       maka:
                1     2    6     24
                120   720  ...   ...
                ...
          (​ Catatan! : Harus menggunakan Thread dalam penghitungan faktorial)
          
- Batu ketiga adalah Onyx. Batu mulia berwarna hitam mengkilat. Pecahkan
  teka-teki berikut!
    1. Buatlah program C ketiga dengan nama "​ 4c.c​ ". Program ini tidak
       memiliki hubungan terhadap program yang lalu.
    2. Pada program ini, Norland diminta mengetahui jumlah file dan
       folder di direktori saat ini dengan command "​ ls | wc -l​ ". Karena sudah belajar
       IPC, Norland mengerjakannya dengan semangat.
       (​ Catatan!​ : Harus menggunakan IPC Pipes)
       
Begitu batu terakhir berhasil didapatkan. Gemuruh yang semakin lama semakin
besar terdengar. Seluruh tempat berguncang dahsyat, tanah mulai merekah. Sebuah
batu yang di atasnya terdapat kotak kayu muncul ke atas dengan sendirinya. 

Sementara batu tadi kembali ke posisinya. Tanah kembali menutup, seolah tidak
pernah ada lubang merekah di atasnya satu detik lalu.

Norland segera memasukkan tiga buah batu mulia Emerald, Amethys, Onyx pada
Peti Kayu. Maka terbukalah Peti Kayu tersebut. Di dalamnya terdapat sebuah harta
karun rahasia. Sampai saat ini banyak orang memburu harta karun tersebut.
Sebelum menghilang, dia menyisakan semua petunjuk tentang harta karun tersebut
melalui tulisan dalam buku catatannya yang tersebar di penjuru dunia. "One Piece
does exist".
```

### Source Code :
### Penjelasan :
