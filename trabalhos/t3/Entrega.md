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
