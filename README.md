# Soal Shift SISOP20 Modul 3
## Kelompok T03
### 1. Muhamad Rifaldi - 05311840000022
### 2. Dimas Pramudya H - 05311840000037

---

## Soal  3
Buatlah sebuah program dari c untuk mengkategorikan file. program ini akan memindahkan file sesuai ekstensinya (tidak case sensitive) ke dalam folder sesuai ekstensinya yan folder hasilnya terdapat di working directory ketika program tersebut dijalankan.
program dapat berjalan dengan 3 mode, yaitu ```-f``` dimana user dapat menambahkan argumen file yang bisa dikategorikan sebanyak yang user inginkan, ```\*``` dimana program bekerja mengkategorikan seluruh file yang ada di working directory, kemudian ```-d``` yaitu bekerja mengkategorikan file yang terdapat pada directory yang kita tentukan, kemudian hasilnya akan disimpan di working directory.

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

### Penjelasan : 

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

### Output :

- Argumen  ```-f```  
Sebelum run :  
![-f](/output/fsebelum.png)  

    Setelah run :  
    ![-f](/output/fsetelah.png)

- Argumen ```\*``` :  
Sebelum run :  
![\*](/output/sebelum.png)  

    Setelah run :  
    ![\*](/output/setelah.png)

- Argumen ```-d``` :  
Sebelum run :  
![-d](/output/dsebelum.png)  

    Setelah run :  
    ![-d](/output/dsetelah.png)

## Soal  4a

Norland adalah seorang penjelajah terkenal. Pada suatu malam Norland menyusuri jalan setapak menuju ke sebuah gua dan mendapati tiga pilar yang pada setiap pilarnya ada sebuah batu berkilau yang tertancap. Batu itu berkilau di kegelapan dan setiap batunya memiliki warna yang berbeda.  

Norland mendapati ada sebuah teka-teki yang tertulis di setiap pilar. Untuk dapat            mengambil batu mulia di suatu pilar, Ia harus memecahkan teka-teki yang ada di             pilar tersebut. Norland menghampiri setiap pilar secara bergantian.  

**Batu mulia pertama​**. Emerald. Batu mulia yang berwarna hijau mengkilat. Pada            batu itu Ia menemukan sebuah kalimat petunjuk. Ada sebuah teka-teki yang berisi:   

1. Buatlah program C dengan nama "​4a.c​", yang berisi program untuk           melakukan perkalian matriks. Ukuran matriks pertama adalah ​4x2​, dan         matriks kedua ​2x5​. Isi dari matriks didefinisikan ​di dalam kodingan. Matriks           nantinya akan berisi angka 1-20 (​tidak perlu​ dibuat filter angka). 
2. Tampilkan matriks hasil perkalian tadi ke layar.  

### Source Code :

````c
#include <stdio.h>
#include <pthread.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>

/*
matA -> Berisi matriks pertama berukuran 4x2
*/
int matA[4][2] = {
  {1, 1},
  {2, 2},
  {1, 1},
  {2, 2}
};

/*
matB -> Berisi matriks kedua berukuran 2x5
*/
int matB[2][5] = {
  {1, 4, 1, 2, 1},
  {1, 2, 1, 4, 1}
};
int matC[4][5];

struct args {
  int i;
  int j;
};

void *kali(void* arg) {
  int i = ((struct args*)arg)->i;
  int j = ((struct args*)arg)->j;

  for (int k = 0; k < 2; k++) {
    matC[i][j] += matA[i][k] * matB[k][j];
  }
}

int main() {

  pthread_t tid[4][5];

  for (int i = 0; i < 4; i++) {
    for (int j = 0; j < 5; j++) {
      struct args *index = (struct args *)malloc(sizeof(struct args));
      index->i = i;
      index->j = j;
      pthread_create(&tid[i][j], NULL, &kali, (void *)index);
    }
  }

  for (int i = 0; i < 4; i++) {
    for (int j = 0; j < 5; j++) {
      pthread_join(tid[i][j], NULL);
    }
  }

  printf("Matriks :\n");
  for (int i = 0; i < 4; i++) {
    for (int j = 0; j < 5; j++) {
      printf("%4d", matC[i][j]);
    }
    printf("\n");
  }

  key_t key = 1337;
  int *value;

  int shmid = shmget(key, sizeof(matC), IPC_CREAT | 0666);
  value = shmat(shmid, NULL, 0);

  int* p = (int *)value;

  memcpy(p, matC, 80);

  shmdt(value);
}

````

### Penjelasan :

- Program ini memiliki beberapa header diantaranya :  

```c
#include <stdio.h>
#include <pthread.h>
#include <sys/ipc.h>
#include <sys/shm.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
```

- ```matA``` Berisi matriks pertama berukuran 4x2

```c
int matA[4][2] = {
  {1, 1},
  {2, 2},
  {1, 1},
  {2, 2}
};
```

- ```matB``` Berisi matriks kedua berukuran 2x5

```c
int matB[2][5] = {
  {1, 4, 1, 2, 1},
  {1, 2, 1, 4, 1}
};
```
- ```int matC[4][5];``` Akan berisi hasil dari perkalian kedua matriks diatas
- ```*kali``` berisi fungsi untuk mengalikan kedua matriks

```c
void *kali(void* arg) {
  int i = ((struct args*)arg)->i;
  int j = ((struct args*)arg)->j;

  for (int k = 0; k < 2; k++) {
    matC[i][j] += matA[i][k] * matB[k][j];
  }
}
```

- Lalu di main function

```c
int main() {

  pthread_t tid[4][5];

  for (int i = 0; i < 4; i++) {
    for (int j = 0; j < 5; j++) {
      struct args *index = (struct args *)malloc(sizeof(struct args));
      index->i = i;
      index->j = j;
      pthread_create(&tid[i][j], NULL, &kali, (void *)index);
    }
  }

  for (int i = 0; i < 4; i++) {
    for (int j = 0; j < 5; j++) {
      pthread_join(tid[i][j], NULL);
    }
  }

  printf("Matriks :\n");
  for (int i = 0; i < 4; i++) {
    for (int j = 0; j < 5; j++) {
      printf("%4d", matC[i][j]);
    }
    printf("\n");
  }

  key_t key = 1337;
  int *value;

  int shmid = shmget(key, sizeof(matC), IPC_CREAT | 0666);
  value = shmat(shmid, NULL, 0);

  int* p = (int *)value;

  memcpy(p, matC, 80);

  shmdt(value);
}
```
### Output :

![4a](/output/4a.png)

## Soal  4a

Norland adalah seorang penjelajah terkenal. Pada suatu malam Norland menyusuri jalan setapak menuju ke sebuah gua dan mendapati tiga pilar yang pada setiap pilarnya ada sebuah batu berkilau yang tertancap. Batu itu berkilau di kegelapan dan setiap batunya memiliki warna yang berbeda.  

Norland mendapati ada sebuah teka-teki yang tertulis di setiap pilar. Untuk dapat            mengambil batu mulia di suatu pilar, Ia harus memecahkan teka-teki yang ada di             pilar tersebut. Norland menghampiri setiap pilar secara bergantian.  

**Batu mulia pertama​**. Emerald. Batu mulia yang berwarna hijau mengkilat. Pada            batu itu Ia menemukan sebuah kalimat petunjuk. Ada sebuah teka-teki yang berisi:   

1. Buatlah program C dengan nama "​4a.c​", yang berisi program untuk           melakukan perkalian matriks. Ukuran matriks pertama adalah ​4x2​, dan         matriks kedua ​2x5​. Isi dari matriks didefinisikan ​di dalam kodingan. Matriks           nantinya akan berisi angka 1-20 (​tidak perlu​ dibuat filter angka). 
2. Tampilkan matriks hasil perkalian tadi ke layar.  

### Source Code :

````c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<sys/types.h>
#include<string.h>
#include<sys/wait.h>

int main() {
  int fd[2];

  pid_t pid;

  pipe(fd);

  pid = fork();
  if (pid == 0) {
    dup2(fd[1], 1);
    close(fd[0]);
    char *argv[] = {"ls", NULL};
    execv("/bin/ls", argv);
  }
  while(wait(NULL) > 0);

  dup2(fd[0], 0);
  close(fd[1]);
  char *argv[] = {"wc", "-l", NULL};
  execv("/usr/bin/wc", argv);

}
````

### Penjelasan :

- Program ini memiliki beberapa header diantaranya :  

```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<sys/types.h>
#include<string.h>
#include<sys/wait.h>
```

- Fungsi berikut menjalankan perintah ```ls``` kemudian informasi tersebut di ***write*** oleh operator ```dup2(fd[1], 1)```

```c
pid = fork();
  if (pid == 0) {
    dup2(fd[1], 1);
    close(fd[0]);
    char *argv[] = {"ls", NULL};
    execv("/bin/ls", argv);
  }
```

- Operator ```dup2(fd[0], 0)``` pada program berikut membaca (read) informasi dari perintah sebelumnya yaitu ```ls``` untuk kemudian di pakai untuk menjalankan perintah ```wc -l```

```c
dup2(fd[0], 0);
  close(fd[1]);
  char *argv[] = {"wc", "-l", NULL};
  execv("/usr/bin/wc", argv);
```
### Output :

![4c](/output/4c.png)
