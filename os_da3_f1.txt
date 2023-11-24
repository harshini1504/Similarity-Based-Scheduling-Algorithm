#include <stdio.h>
#include <unistd.h>
#include <stdbool.h>
#include <stdlib.h>
#include <sys/wait.h>

#define MAX_PROCESSES 10
#define MAX_RESOURCES 3

typedef struct {
  int cpuCore;
} Process;

float calculateSimilarity(int resources[MAX_PROCESSES][MAX_RESOURCES], int process1, int process2) {
  int commonResources = 0;
  int totalResources1 = 0, totalResources2 = 0;

  for (int i = 0; i < MAX_RESOURCES; i++) {
    commonResources += (resources[process1][i] < resources[process2][i]) ? resources[process1][i] : resources[process2][i];
    totalResources1 += resources[process1][i];
    totalResources2 += resources[process2][i];
  }

  return (float)commonResources / (float)(totalResources1 + totalResources2);
}
void allocateResources(int resources[MAX_PROCESSES][MAX_RESOURCES], int numProcesses, int numCores, Process* coreAllocation) {
  bool allocated[MAX_PROCESSES] = {false}; 
  int processesAllocated = 0;
  int core1Processes = 0;
  int core2Processes = 0;
 while (processesAllocated < numProcesses) {
    int maxSimilarityProcess = -1;
    float maxSimilarity = 0.0;

  for (int i = 0; i < numProcesses; i++) {
      if (!allocated[i]) {
        for (int j = 0; j < numProcesses; j++) {
          if (i != j && !allocated[j]) {
            float similarity = calculateSimilarity(resources, i, j);
            if (similarity > maxSimilarity) {
              maxSimilarity = similarity;
              maxSimilarityProcess = j;
            }
          }
        }
      }
    }
    if (core1Processes < numCores / 2) {
      coreAllocation[maxSimilarityProcess].cpuCore = core1Processes + 1;
      core1Processes++;
    } else if (core2Processes < numCores / 2) {
      coreAllocation[maxSimilarityProcess].cpuCore = core2Processes + 1;
      core2Processes++;
    }

    allocated[maxSimilarityProcess] = true;
    processesAllocated++;
  }
}

int main() {
  int resources[MAX_PROCESSES][MAX_RESOURCES];
  int numProcesses, numCores;
  Process coreAllocation[MAX_PROCESSES]; // Declaration for coreAllocation array

  printf("Enter the number of processes: ");
  scanf("%d", &numProcesses);

  if (numProcesses <= 0 || numProcesses > MAX_PROCESSES) {
    printf("Invalid number of processes. Maximum supported processes: %d\n", MAX_PROCESSES);
    return 1;
  }

  printf("Enter the number of cores: ");
  scanf("%d", &numCores);

  if (numCores <= 0 || numCores > numProcesses || numCores > MAX_PROCESSES || numCores % 2 != 0) {
    printf("Invalid number of cores. Maximum supported cores: %d (even number)\n", numProcesses);
    return 1;
  }
  for (int i = 0; i < numProcesses; i++) {
    printf("Enter resources for Process %d:\n", i);
    for (int j = 0; j < MAX_RESOURCES; j++) {
      printf("Resource %d: ", j + 1);
      scanf("%d", &resources[i][j]);
    }
  }
  allocateResources(resources, numProcesses, numCores, coreAllocation);
  for (int i = 0; i < numCores; i++) {
    printf("Core %d: ", i);
    for (int j = 0; j < numProcesses; j++) {
      if (coreAllocation[j].cpuCore == i) {
        printf("PID%d ", j);
      }
    }
    printf("\n");
  }
  for (int i = 0; i < numProcesses; i++) {
    pid_t pid = fork();

    if (pid < 0) {
      fprintf(stderr, "Failed to fork process\n");
      return 1;
    } else if (pid == 0) {
      printf("Child process %d: PID=%d, Parent PID=%d, CPU core=%d\n", i, getpid(), getppid(), coreAllocation[i].cpuCore);
      exit(0);
    }
  }
  for (int i = 0; i < numProcesses; i++) {
    wait(NULL);
  }

  return 0;
}
