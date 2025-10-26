#include <stdio.h>
#include <stdlib.h>
#include <time.h>
#include <stdbool.h>

typedef struct Customer {
    int id;
    int arrival_time;
    int service_duration;
    int time_served;
    int wait_time;
} Customer;

typedef struct Node {
    Customer customer;
    struct Node *next;
} Node;

typedef struct Queue {
    Node *front;
    Node *rear;
    int size;
} Queue;

typedef struct Teller {
    int id;
    bool is_free;
    Customer *current_customer;
    int busy_time;
} Teller;

void initialize_queue(Queue *q) {
    q->front = NULL;
    q->rear = NULL;
    q->size = 0;
}

bool is_empty(Queue *q) {
    return q->front == NULL;
}

void enqueue(Queue *q, Customer c) {
    Node *new_node = (Node *)malloc(sizeof(Node));
    if (new_node == NULL) {
        printf("Memory allocation failed!\n");
        exit(EXIT_FAILURE);
    }
    new_node->customer = c;
    new_node->next = NULL;

    if (is_empty(q)) {
        q->front = new_node;
        q->rear = new_node;
    } else {
        q->rear->next = new_node;
        q->rear = new_node;
    }
    q->size++;
}

Customer dequeue(Queue *q) {
    if (is_empty(q)) {
        exit(EXIT_FAILURE);
    }

    Node *temp = q->front;
    Customer c = temp->customer;

    q->front = q->front->next;
    if (q->front == NULL) {
        q->rear = NULL;
    }

    free(temp);
    q->size--;
    return c;
}

int main() {
    srand(time(NULL));

    int max_time;
    int num_tellers;
    int arrival_rate;

    printf("--- Bank Queue Simulation Setup ---\n");

    printf("Enter total simulation time (minutes, e.g., 60): ");
    if (scanf("%d", &max_time) != 1 || max_time <= 0) {
        printf("Invalid time. Exiting.\n");
        return 1;
    }

    printf("Enter number of tellers (e.g., 3): ");
    if (scanf("%d", &num_tellers) != 1 || num_tellers <= 0) {
        printf("Invalid teller count. Exiting.\n");
        return 1;
    }

    printf("Enter customer arrival rate (percent per minute, 0-100, e.g., 30): ");
    if (scanf("%d", &arrival_rate) != 1 || arrival_rate < 0 || arrival_rate > 100) {
        printf("Invalid arrival rate. Exiting.\n");
        return 1;
    }


    Queue customer_queue;
    initialize_queue(&customer_queue);

    Teller tellers[num_tellers];
    for (int i = 0; i < num_tellers; i++) {
        tellers[i].id = i + 1;
        tellers[i].is_free = true;
        tellers[i].current_customer = NULL;
        tellers[i].busy_time = 0;
    }

    int customer_counter = 0;
    int customers_served = 0;
    long total_wait_time = 0;
    int max_queue_length = 0;

    printf("\n--- Bank Queue Simulation Start ---\n");
    printf("Tellers: %d | Max Time: %d min | Arrival Rate: %d%%/min\n\n",
           num_tellers, max_time, arrival_rate);


    for (int current_time = 0; current_time < max_time; current_time++) {
        printf("[Time %02d min] ", current_time);
        bool activity_this_minute = false;

        if ((rand() % 100) < arrival_rate) {
            customer_counter++;
            Customer new_customer;
            new_customer.id = customer_counter;
            new_customer.arrival_time = current_time;
            new_customer.service_duration = (rand() % 10) + 1;
            new_customer.time_served = 0;
            new_customer.wait_time = 0;

            enqueue(&customer_queue, new_customer);
            printf("Customer %d arrives (Service: %d min) -> ENQUEUED. ",
                   new_customer.id, new_customer.service_duration);
            activity_this_minute = true;
        }

        if (customer_queue.size > max_queue_length) {
            max_queue_length = customer_queue.size;
        }

        for (int i = 0; i < num_tellers; i++) {
            if (!tellers[i].is_free) {
                tellers[i].current_customer->time_served++;
                tellers[i].busy_time++;

                if (tellers[i].current_customer->time_served >= tellers[i].current_customer->service_duration) {
                    customers_served++;

                    total_wait_time += tellers[i].current_customer->wait_time;

                    printf("Teller %d finishes Customer %d (Wait: %d min). ",
                           tellers[i].id, tellers[i].current_customer->id, tellers[i].current_customer->wait_time);

                    free(tellers[i].current_customer);
                    tellers[i].current_customer = NULL;
                    tellers[i].is_free = true;
                    activity_this_minute = true;
                }
            }
        }

        for (int i = 0; i < num_tellers; i++) {
            if (tellers[i].is_free && !is_empty(&customer_queue)) {
                Customer next_customer_data = dequeue(&customer_queue);

                tellers[i].current_customer = (Customer *)malloc(sizeof(Customer));
                if (tellers[i].current_customer == NULL) {
                    printf("Memory allocation failed!\n");
                    exit(EXIT_FAILURE);
                }
                *tellers[i].current_customer = next_customer_data;

                tellers[i].current_customer->wait_time = current_time - tellers[i].current_customer->arrival_time;

                tellers[i].is_free = false;

                printf("Teller %d starts serving Customer %d (Wait: %d min). ",
                       tellers[i].id, tellers[i].current_customer->id, tellers[i].current_customer->wait_time);
                activity_this_minute = true;
            }
        }

        if (!activity_this_minute) {
            printf("No change.\n");
        } else {
            printf("\n");
        }
    }

    printf("\n--- Simulation Complete ---\n");

    printf("\n--- PERFORMANCE METRICS ---\n");

    printf("Total Customers Served: %d\n", customers_served);

    if (customers_served > 0) {
        double avg_wait_time = (double)total_wait_time / customers_served;
        printf("Average Wait Time: %.2f minutes\n", avg_wait_time);
    } else {
        printf("Average Wait Time: N/A (No customers served)\n");
    }

    printf("Maximum Queue Length: %d customers\n", max_queue_length);

    int total_possible_time = num_tellers * max_time;
    int total_busy_time = 0;
    for (int i = 0; i < num_tellers; i++) {
        total_busy_time += tellers[i].busy_time;
    }

    double utilization_rate = ((double)total_busy_time / total_possible_time) * 100;
    printf("Teller Utilization Rate: %.2f%%\n", utilization_rate);

    int remaining_in_service = 0;
    for (int i = 0; i < num_tellers; i++) {
        if (!tellers[i].is_free) {
            remaining_in_service++;
            free(tellers[i].current_customer);
        }
    }
    int remaining_customers = customer_queue.size + remaining_in_service;
    printf("Customers Remaining (unserved in Queue/at Teller): %d\n", remaining_customers);

    while (!is_empty(&customer_queue)) {
        dequeue(&customer_queue);
    }

    return 0;
}
