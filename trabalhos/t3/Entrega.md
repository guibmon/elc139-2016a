	Nome: Guilherme Bolzan Monteiro
	ELC139 - Programação Paralela

	Parte 1: Pthreads
1) 
A etapa de particionamento se dá na definição da struct dotdata_t ao termos dois vetores (a e b):

        typedef struct 
        {
                double *a;
                double *b;
                (...)
        }
   
   
A comunicação se dá quando são criadas as threads:

        for (i = 0; i < nthreads; i++) {
                pthread_create(&threads[i], &attr, dotprod_worker, (void *) i);
        }
  
A aglomeração se dá quando são feitas as somas parciais em cada thread:

        for (k = 0; k < dotdata.repeat; k++) {
                mysum = 0.0;
                for (i = start; i < end ; i++)  {
                        mysum += (a[i] * b[i]);
                }
        }

O mapeamento se dá quando as threads são alocadas e inicializadas:

        threads = (pthread_t *) malloc(nthreads * sizeof(pthread_t));
        pthread_mutex_init(&mutexsum, NULL);

2), 3) e 4) Tabela com valores de threads, tamanho dos vetores, número de repetições e tempo médio em segundos:

Nº threads | Tamanho vetores | Repetições | Tempo médio(em s)
---------- | --------------- | ---------- | -----------------
1          | 1000000         | 2000       | 6,415
2          | 500000          | 2000       | 4,137
4          | 250000          | 2000       | 3,087
8          | 125000          | 2000       | 3,127
1          | 2000000         | 2000       | 12,818
2          | 1000000         | 2000       | 8,220
4          | 500000          | 2000       | 6,308
8          | 250000          | 2000       | 6,320
1          | 400000          | 5000       | 6,557
2          | 200000          | 5000       | 4,080
4          | 100000          | 5000       | 2,956
8          | 50000           | 5000       | 2,811
16         | 25000           | 5000       | 2,787


5) O programa pthreads_dotprod apresenta os seguintes comandos para fazer a soma de cada thread com a soma parcial total da 
execução:

        pthread_mutex_lock (&mutexsum);
        dotdata.c += mysum;
        pthread_mutex_unlock (&mutexsum);

Isso garante que apenas uma thread fará a soma por vez. Uma thread tranca a instrução da soma(lock) para que outras threads
não a usem ao mesmo tempo, faz a soma e destranca essa instrução para as demais threads. Se uma thread tentar chegar na soma
quando o lock está ativo, ela ficará parada até a thread que está fazendo a soma dar o unlock.
   
Já o programa pthreads_dotprod2 apresenta apenas o seguinte para as somas parciais:
  
        dotdata.c += mysum;
   
Isso não garante que apenas uma thread faça a soma parcial por vez, o que pode gerar um resultado errado ao final da execução.
Cabe ressaltar que nem sempre ocorrerá essa discrepância, mas sem o uso do lock e unlock estaremos correndo esse risco.


	Parte 2: OpenMP
1) O programa que fiz dá o seguinte erro:
libgomp: Invalid value for environment variable OMP_NUM_THREADS
O que eu fiz é o seguinte:

        #include <stdio.h>
        #include <stdlib.h>
        #include <omp.h>
        #include <time.h>

        typedef struct 
         {
                double *a;
                double *b;
                double c; 
                int wsize;
                int repeat; 
        } dotdata_t;

        // Variaveis globais, acessiveis por todas threads
        dotdata_t dotdata;
        omp_lock_t mutexsum;
        int OMP_NUM_THREADS;

        void omp_set_num_threads(int nthreads){
	        OMP_NUM_THREADS = nthreads;
        }

        void *dotprod_worker(void *arg)
        {
                int i, k;
                long offset = (long) arg;
                double *a = dotdata.a;
                double *b = dotdata.b;     
                int wsize = dotdata.wsize;
                int start = offset*wsize;
                int end = start + wsize;
                double mysum;

                for (k = 0; k < dotdata.repeat; k++) {
                        mysum = 0.0;
                        for (i = start; i < end ; i++)  {
                                mysum += (a[i] * b[i]);
                        }
                }
                omp_set_lock (&mutexsum);
                dotdata.c += mysum;
                omp_unset_lock (&mutexsum);
                }

        void dotprod_threads(int OMP_NUM_THREADS)
        {
                int i;
                omp_init_lock(&mutexsum);
                #pragma omp parallel for
                for (i = 0; i < OMP_NUM_THREADS; i++) {
                        dotprod_worker;
                }
   	}

        long wtime()
        {
                struct timeval t;
                gettimeofday(&t, NULL);
                return t.tv_sec*1000000 + t.tv_usec;
        }
        
        void fill(double *a, int size, double value)
        {  
                int i;
                for (i = 0; i < size; i++) {
                a[i] = value;
                }
        }

        int main(int argc, char **argv)
        {
                int nthreads, wsize, repeat;
                long start_time, end_time;

                if ((argc != 4)) {
                        printf("Uso: %s <nthreads> <worksize> <repetitions>\n", argv[0]);
                        exit(EXIT_FAILURE);
                }

                nthreads = atoi(argv[1]); 
                wsize = atoi(argv[2]);  // worksize = tamanho do vetor de cada thread
                repeat = atoi(argv[3]); // numero de repeticoes dos calculos (para aumentar carga)

                omp_set_num_threads(nthreads);
                dotdata.a = (double *) malloc(wsize*nthreads*sizeof(double));
                fill(dotdata.a, wsize*nthreads, 0.01);
                dotdata.b = (double *) malloc(wsize*nthreads*sizeof(double));
                fill(dotdata.b, wsize*nthreads, 1.0);
                dotdata.c = 0.0;
                dotdata.wsize = wsize;
                dotdata.repeat = repeat;

                start_time = wtime();
                dotprod_threads(OMP_NUM_THREADS);
                end_time = wtime();

                printf("%f\n", dotdata.c);
                printf("%d thread(s), %ld usec\n", OMP_NUM_THREADS, (long) (end_time - start_time));
                fflush(stdout);

                free(dotdata.a);
                free(dotdata.b);

                return EXIT_SUCCESS;
        }
        
        
	Referências: 
	https://www.dcc.fc.up.pt/~ricroc/aulas/0708/ppd/apontamentos/fundamentos.pdf
	https://www.sourceware.org/pthreads-win32/manual/pthread_mutex_init.html
	www.ibm.com/developerworks/br/aix/library/au-aix-openmp-framework/
	https://www.dartmouth.edu/~rc/classes/intro_openmp/compile_run.html
