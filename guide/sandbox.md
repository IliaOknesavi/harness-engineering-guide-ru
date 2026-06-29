---
author: Nexu
---

# Песочница

> **Ключевая идея:** Агент с доступом к командной оболочке может выполнить `rm -rf /`. Песочница (sandbox) — это то, что отличает полезного ассистента-кодера от источника проблем. Модель должна ощущать себя ничем не ограниченной; среда исполнения — наоборот, ограниченной по максимуму.

## Зачем нужна песочница?

Когда агент запускает код, устанавливает пакеты или выполняет команды оболочки, он действует с реальными системными привилегиями. Без изоляции единственный сгаллюцинированный `curl ... | bash` может выкачать данные, установить вредоносное ПО или уничтожить хост. Песочница ограничивает радиус поражения — даже если агент сделает что-то опасное, ущерб будет локализован.

Три вектора угроз:
1. **Утечка данных (data exfiltration)** — агент читает секреты и отправляет их на внешний сервер
2. **Разрушительные операции** — агент удаляет файлы, портит базы данных, изменяет системную конфигурацию
3. **Эскалация привилегий (privilege escalation)** — агент выходит за пределы своей песочницы, получая доступ к хосту

Промышленная песочница закрывает все три вектора одновременно.

## Настройка песочницы на Docker

Docker — самая распространённая песочница для однопользовательских (single-tenant) развёртываний агентов. Главное — запускать с ограничивающими настройками по умолчанию и ослаблять лишь то, что явно необходимо:

```dockerfile
# Dockerfile.sandbox
FROM python:3.12-slim

# Non-root user — never run agents as root
RUN useradd -m -s /bin/bash agent
WORKDIR /workspace

# Install common tools (locked versions, no auto-update)
RUN pip install --no-cache-dir \
    ruff==0.4.4 \
    pytest==8.2.0 \
    httpx==0.27.0

# Drop all capabilities, agent gets only what's listed
USER agent
```

Вызов `docker run` важнее самого Dockerfile — именно здесь и применяются настоящие ограничения:

```python
import subprocess
import tempfile
import json
from pathlib import Path

class DockerSandbox:
    """Execute agent commands inside a restricted Docker container."""

    def __init__(
        self,
        image: str = "agent-sandbox:latest",
        workspace: str | None = None,
        timeout: int = 30,
        memory_limit: str = "512m",
        network: bool = False,
    ):
        self.image = image
        self.workspace = workspace or tempfile.mkdtemp(prefix="agent-")
        self.timeout = timeout
        self.memory_limit = memory_limit
        self.network = network

    def execute(self, command: str) -> dict:
        """Run a command in the sandbox and return stdout/stderr/exit code."""
        docker_cmd = [
            "docker", "run",
            "--rm",                              # Auto-cleanup
            "--user", "1000:1000",               # Non-root
            "--memory", self.memory_limit,       # OOM protection
            "--cpus", "1.0",                     # CPU limit
            "--pids-limit", "100",               # Fork bomb protection
            "--read-only",                       # Read-only root filesystem
            "--tmpfs", "/tmp:size=100m",         # Writable temp space
            "--tmpfs", "/workspace:size=200m",   # Writable workspace
            "--security-opt", "no-new-privileges",
            "--cap-drop", "ALL",                 # Drop all Linux capabilities
        ]

        # Mount workspace files as read-only input
        if Path(self.workspace).exists():
            docker_cmd.extend([
                "-v", f"{self.workspace}:/input:ro"
            ])

        # Network isolation (default: no network)
        if not self.network:
            docker_cmd.extend(["--network", "none"])

        docker_cmd.extend([self.image, "bash", "-c", command])

        try:
            result = subprocess.run(
                docker_cmd,
                capture_output=True,
                text=True,
                timeout=self.timeout,
            )
            return {
                "stdout": result.stdout[-10_000:],  # Truncate large output
                "stderr": result.stderr[-5_000:],
                "exit_code": result.returncode,
            }
        except subprocess.TimeoutExpired:
            return {
                "stdout": "",
                "stderr": f"Command timed out after {self.timeout}s",
                "exit_code": -1,
            }
```

Критически важные флаги:
- `--read-only` делает корневую файловую систему неизменяемой. Агент не может изменять системные бинарники или устанавливать постоянные бэкдоры.
- `--network none` запрещает любой сетевой доступ. Агент не может выкачать данные или загрузить вредоносную нагрузку.
- `--cap-drop ALL` убирает Linux-capabilities. Никаких `ptrace`, `mount`, `chown`.
- `--tmpfs` предоставляет записываемое временное пространство, которое исчезает при завершении контейнера.

## MicroVM на Firecracker для мультитенантности

Docker обеспечивает изоляцию на уровне процессов, чего достаточно для однопользовательского сценария. Мультитенантность (несколько недоверенных пользователей делят один хост) требует более жёстких гарантий — побег из контейнера затрагивает всех арендаторов. Эту проблему решают microVM на базе Firecracker:

```
┌──────────────────────────────────────┐
│  Host Machine                        │
│  ┌────────────┐  ┌────────────┐     │
│  │ MicroVM    │  │ MicroVM    │     │
│  │ (User A)   │  │ (User B)   │     │
│  │ ┌────────┐ │  │ ┌────────┐ │     │
│  │ │ Agent  │ │  │ │ Agent  │ │     │
│  │ └────────┘ │  │ └────────┘ │     │
│  └────────────┘  └────────────┘     │
│  Firecracker VMM                     │
│  KVM hypervisor boundary             │
└──────────────────────────────────────┘
```

Каждая microVM загружается примерно за ~125 мс с минимальным ядром Linux. Граница гипервизора означает, что эксплойт ядра внутри ВМ не сможет добраться до хоста или других ВМ:

```python
import json
import socket

class FirecrackerSandbox:
    """Manage a Firecracker microVM for agent execution."""

    def __init__(self, socket_path: str, kernel: str, rootfs: str):
        self.socket_path = socket_path
        self.kernel = kernel
        self.rootfs = rootfs

    def configure(self, vcpus: int = 1, mem_mb: int = 256):
        """Configure the microVM resources."""
        self._api_call("PUT", "/machine-config", {
            "vcpu_count": vcpus,
            "mem_size_mib": mem_mb,
        })
        self._api_call("PUT", "/boot-source", {
            "kernel_image_path": self.kernel,
            "boot_args": "console=ttyS0 reboot=k panic=1 pci=off",
        })
        self._api_call("PUT", "/drives/rootfs", {
            "drive_id": "rootfs",
            "path_on_host": self.rootfs,
            "is_root_device": True,
            "is_read_only": False,
        })

    def start(self):
        """Boot the microVM."""
        self._api_call("PUT", "/actions", {"action_type": "InstanceStart"})

    def stop(self):
        """Shut down the microVM."""
        self._api_call("PUT", "/actions", {"action_type": "SendCtrlAltDel"})

    def _api_call(self, method: str, path: str, body: dict):
        """Make an API call to the Firecracker socket."""
        sock = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        sock.connect(self.socket_path)
        payload = json.dumps(body)
        request = (
            f"{method} {path} HTTP/1.1\r\n"
            f"Content-Type: application/json\r\n"
            f"Content-Length: {len(payload)}\r\n"
            f"\r\n{payload}"
        )
        sock.sendall(request.encode())
        response = sock.recv(4096).decode()
        sock.close()
        return response
```

## Принудительное применение разрешений на уровне ОС

Помимо изоляции контейнера, применяйте разрешения внутри песочницы с помощью модулей безопасности Linux:

```bash
# seccomp profile — restrict system calls
# sandbox-seccomp.json
{
    "defaultAction": "SCMP_ACT_ERRNO",
    "syscalls": [
        {
            "names": ["read", "write", "open", "close", "stat", "fstat",
                       "mmap", "mprotect", "munmap", "brk", "execve",
                       "access", "pipe", "dup2", "fork", "wait4", "exit_group"],
            "action": "SCMP_ACT_ALLOW"
        }
    ]
}
```

Подключите его флагом `--security-opt seccomp=sandbox-seccomp.json` в команде docker run. Агент сможет читать, писать и выполнять — но не монтировать файловые системы, не загружать модули ядра и не создавать «сырые» сокеты.

## Сетевая изоляция

`--network none` блокирует весь трафик, но некоторым агентам нужен *ограниченный* сетевой доступ (например, для `pip install` или вызовов API). Используйте сетевую политику, чтобы разрешить лишь конкретные адреса назначения:

```python
NETWORK_ALLOWLIST = [
    "pypi.org:443",
    "files.pythonhosted.org:443",
    "api.openai.com:443",
]

def create_sandbox_network():
    """Create a Docker network with egress restrictions via iptables."""
    subprocess.run([
        "docker", "network", "create",
        "--driver", "bridge",
        "--opt", "com.docker.network.bridge.enable_icc=false",
        "agent-sandbox-net",
    ], check=True)

    # Allow only specific destinations
    for target in NETWORK_ALLOWLIST:
        host, port = target.split(":")
        subprocess.run([
            "iptables", "-A", "DOCKER-USER",
            "-d", host,
            "-p", "tcp", "--dport", port,
            "-j", "ACCEPT",
        ], check=True)

    # Drop everything else
    subprocess.run([
        "iptables", "-A", "DOCKER-USER", "-j", "DROP",
    ], check=True)
```

## Ограничения файловой системы

Накладывайте несколько уровней контроля файловой системы:

```python
def build_volume_mounts(workspace: str, readonly_dirs: list[str]) -> list[str]:
    """Construct Docker volume mount arguments."""
    mounts = [
        # Agent workspace — read-write, but scoped
        f"-v {workspace}:/workspace:rw",
        # Temp space — in-memory, size-limited
        "--tmpfs /tmp:size=100m,noexec",
    ]
    # Read-only reference directories
    for d in readonly_dirs:
        mounts.append(f"-v {d}:{d}:ro")
    return mounts

# Example: agent can read source code but only write to /workspace
mounts = build_volume_mounts(
    workspace="/tmp/agent-work-abc123",
    readonly_dirs=["/opt/project/src", "/opt/project/tests"],
)
```

Флаг `noexec` на `/tmp` не даёт агенту записать скрипты во временный каталог и запустить их — это распространённый приём побега.

## Собираем всё вместе

Промышленный исполнитель-песочница объединяет все уровни:

```python
class ProductionSandbox:
    """Full-featured sandbox with layered security."""

    def __init__(self, config: dict):
        self.docker = DockerSandbox(
            image=config["image"],
            timeout=config.get("timeout", 30),
            memory_limit=config.get("memory", "512m"),
            network=config.get("network", False),
        )

    def run_tool(self, tool_name: str, command: str) -> str:
        """Execute a tool command in the sandbox."""
        # Log every execution for audit
        log_entry = {"tool": tool_name, "command": command}
        audit_log.append(log_entry)

        result = self.docker.execute(command)

        if result["exit_code"] != 0:
            return f"Error (exit {result['exit_code']}):\n{result['stderr']}"
        return result["stdout"]
```

## Типичные ошибки

- **Запуск под root** — самая частая ошибка при настройке песочницы. Даже внутри Docker root может изменять файловую систему контейнера, устанавливать пакеты и потенциально эксплуатировать уязвимости ядра. Всегда используйте непривилегированного (non-root) пользователя.
- **Забыли `--network none`** — без явного запрета сети контейнер наследует сеть хоста. Агент сможет `curl`-ом отправить секреты на любой сервер.
- **Долгоживущие контейнеры** — если контейнер песочницы сохраняется между вызовами, агент может накапливать состояние, устанавливать бэкдоры или настраивать cron-задания. По умолчанию используйте эфемерные контейнеры (`--rm`).
- **Доверие к выводу агента** — `cat /etc/passwd` в песочнице всё равно вернёт реальные данные, если файл смонтирован. Монтируйте только то, что нужно, и монтируйте в режиме «только чтение».

## Дополнительное чтение

- [Firecracker: Lightweight Virtualization](https://firecracker-microvm.github.io/) — движок microVM, на котором работают AWS Lambda и Fly.io
- [Docker Security Best Practices](https://docs.docker.com/engine/security/) — capabilities, seccomp и профили AppArmor
- [E2B: Open Source Sandbox](https://e2b.dev/) — облачный сервис-песочница, специально созданный для ИИ-агентов
