# Understanding Memory and CPU Units in Kubernetes

### Memory

Memory in Kubernetes is typically expressed in bytes. The most common units are:

	•	Mi (Mebibytes): 1 MiB = 2^20 bytes = 1,048,576 bytes
	•	Gi (Gibibytes): 1 GiB = 2^30 bytes = 1,073,741,824 bytes
	•	Ki (Kibibytes): 1 KiB = 2^10 bytes = 1,024 bytes

### CPU

CPU resources are measured in cores. The units are:

	•	m (millicores): 1 core = 1000 millicores
	•	cores: Fractional values like 0.5 or 1.5 cores

Example Calculations

	•	Memory:
	•	64Mi means 64 Mebibytes.
	•	In bytes: 64 MiB = 64 * 2^20 bytes = 64 * 1,048,576 bytes = 67,108,864 bytes.
	•	CPU:
	•	250m means 250 millicores.
	•	In cores: 250 millicores = 250 / 1000 cores = 0.25 cores.
