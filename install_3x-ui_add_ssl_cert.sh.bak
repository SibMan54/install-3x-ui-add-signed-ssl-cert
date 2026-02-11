#!/bin/bash

# Устанавливаем 3x-ui панель для VLESS и сертификаты на 10 лет

##### COLOR #####

# Определение цветов
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[1;36m'
NC='\033[0m' # Сброс цвета

# Значки
CHECK_MARK="${GREEN}✅${NC}"
WARNING="${YELLOW}⚠${NC}"
CROSS="${RED}❌${NC}"

# Функция для вывода успешных сообщений
success_message() {
    echo -e "${CHECK_MARK} ${GREEN}$1${NC}"
}

# Функция для вывода предупреждений
warning_message() {
    echo -e "${WARNING} ${YELLOW}$1${NC}"
}

# Функция для вывода ошибок
error_message() {
    echo -e "${CROSS} ${RED}$1${NC}"
}

# Функция для вывода запросов
info_message() {
    echo -e "${BLUE}$1${NC}"
}

##### END COLOR #####

# Проверяем, что скрипт выполняется с правами root
if [ "$EUID" -ne 0 ]; then
  error_message "Пожалуйста, запускайте этот скрипт с правами root."
  exit 1
fi

# Установка 3X-UI
if ! command -v x-ui &> /dev/null; then
    if bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh); then
        success_message "3X-UI установлен."
    else
        error_message "Ошибка установки 3X-UI панели."
        exit 1
    fi
else
    success_message "3X-UI уже установлен."
fi

# Генерация сертификата
ssl_detected=$(grep -a 'webCertFile' /etc/x-ui/x-ui.db)
if [ -n "$ssl_detected" ]; then  # Check if the variable is non-empty
    success_message "SSL уже встроин в 3X-UI панель"
else
    echo ""
    read -p "$(echo -e "${YELLOW}Вы хотите сгенерировать-подписать SSL сертификат и встроить его в 3X-UI ? (y/n): ${NC}")" answer
    if [[ "$answer" == "y" ]]; then
      if bash <(curl -Ls https://raw.githubusercontent.com/SibMan54/install-3x-ui-add-signed-ssl-cert/refs/heads/main/3x-ui-autossl.sh); then
        success_message "SSL сертификат успешно сгенерирован и встроин в 3X-UI панель."
      else
        error_message "Ошибка при генерации SSL сертификата."
        exit 1
      fi
    else
        warning_message "SSL сертификат НЕ сгенерирован."
    fi
fi


##### FIREWALL #####

# Функция для извлечения порта 3x-UI
get_3x_ui_port() {
    PORT=$(sudo x-ui settings | grep -i 'port' | grep -oP '\d+')
    if [[ -z "$PORT" ]]; then
        warning_message "Не удалось автоматически определить порт 3x-UI."
        read -p "$(echo -e "${YELLOW}Введите номер порта 3x-UI панели:${NC}")"
        read -r PORT
    fi
    echo "$PORT"
}

# Функция для извлечения порта SSH
get_ssh_port() {
    # SSH_PORT=$(grep -i "^Port " /etc/ssh/sshd_config | awk '{print $2}')
    SSH_PORT=$(awk '$1 == "Port" {print $2; exit}' /etc/ssh/sshd_config)
    if [[ -z "$SSH_PORT" ]]; then
        SSH_PORT=22 # Используем порт по умолчанию
    fi
    echo "$SSH_PORT"
}

# Функция для добавления порта в список разрешённых
add_port_to_ufw() {
    local PORT=$1
    ufw allow "$PORT"/tcp > /dev/null 2>&1
    success_message "Порт $PORT добавлен в список разрешённых (или уже был добавлен)."
}

# Проверяем статус UFW
ufw_status=$(ufw status | grep -i "Status:" | awk '{print $2}')

if [[ "$ufw_status" == "active" ]]; then
    success_message "Firewall уже активен."

    # Извлекаем порт SSH и добавляем его
    SSH_PORT=$(get_ssh_port)
    add_port_to_ufw "$SSH_PORT"

    # Извлекаем порт 3x-UI и добавляем его
    PORT=$(get_3x_ui_port)
    add_port_to_ufw "$PORT"

    # Добавляем порт 443
    add_port_to_ufw 443

    # Применяем изменения
    ufw reload > /dev/null 2>&1
    ufw status numbered
else
    echo ""
    read -p "$(echo -e "${YELLOW}Вы хотите активировать Firewall? (y/n): ${NC}")" answer
    if [[ "$answer" == "y" ]]; then
        # Активируем Firewall
        echo "y" | ufw enable > /dev/null 2>&1
        success_message "Firewall активирован."

        # Извлекаем порт SSH и добавляем его
        SSH_PORT=$(get_ssh_port)
        add_port_to_ufw "$SSH_PORT"

        # Извлекаем порт 3x-UI и добавляем его
        PORT=$(get_3x_ui_port)
        add_port_to_ufw "$PORT"

        # Добавляем порт 443
        add_port_to_ufw 443

        # Применяем изменения
        ufw reload > /dev/null 2>&1
        ufw status numbered
    else
        warning_message "Firewall не активирован."
    fi
fi

##### END FIREWALL #####


# Проверка и установка SpeedTest
echo ""
if command -v speedtest > /dev/null 2>&1; then
    success_message "Speedtest CLI уже установлен."
else
    read -p "$(echo -e "${YELLOW}Установить SpeedTest ? (y/n): ${NC}")" answer
    if [[ "$answer" == "y" ]]; then
        # Установка Speedtest CLI
        if bash <(curl -Ls https://packagecloud.io/install/repositories/ookla/speedtest-cli/script.deb.sh) && apt install -y speedtest-cli; then
            rm -f /etc/apt/sources.list.d/ookla_speedtest-cli.list
            success_message "Speedtest CLI успешно установлен."
        else
            error_message "Ошибка установки Speedtest CLI."
            exit 1
        fi
    else
        warning_message "Speedtest CLI НЕ установлен."
    fi
fi


echo ""

# Финальное сообщение
echo "============================================================================="
if [[ -f /etc/ssl/certs/3x-ui-public.key ]]; then
    info_message " Установка завершена, SSL-сертификат сгенерирован и прописан в панель 3X-UI"
    info_message " Для применения изменений необходимо перезагрузить панель, выполнив команду:"
    echo -e "${CYAN}   sudo x-ui затем вводим 13 и жмем Enter ${NC}"
else
    warning_message " Установка 3X-UI панели завершена, вход в панель не защищен!"
fi
echo "============================================================================="
