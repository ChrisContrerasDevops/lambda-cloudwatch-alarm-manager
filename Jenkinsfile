#!groovy
import java.util.UUID

/*
 * Jenkins Pipeline: Lambda CloudWatch Error Alarms
 *
 * Uso:
 * 1. Copia este Jenkinsfile en un Pipeline de Jenkins.
 * 2. Cambia solo los valores de la sección CONFIGURACIÓN.
 * 3. Ejecuta el pipeline.
 */

// ================================================================
// CONFIGURACIÓN EDITABLE POR EL USUARIO
// ================================================================

// ID de la credencial AWS guardada en Jenkins Credentials.
// Ejemplos: 'jenkins', 'jenkins-org', 'aws-prod', etc.
def AWS_CREDENTIALS_ID = 'TU_CREDENTIAL_ID_DE_JENKINS'

// Región AWS donde se revisarán las Lambdas y donde se crearán los SNS Topics/Alarmas.
def AWS_REGION = 'us-east-1'

// Nombre del rol que Jenkins debe asumir en cada cuenta destino.
// El ARN final se arma así: arn:aws:iam::<ACCOUNT_ID>:role/<AWS_ROLE_NAME>
def AWS_ROLE_NAME = 'lambda-alarmas-sns'

// Correo que recibirá las notificaciones de alarmas SNS.
def NOTIFICATION_EMAIL = 'lambda.notificaciones@tudominio.cl'

// Topic SNS central/opcional para notificar que el proceso terminó por cuenta.
// Si no quieres notificación de proceso, deja este valor vacío: ''
def PROCESS_NOTIFICATION_TOPIC_ARN = 'arn:aws:sns:us-east-1:TU_ACCOUNT_ID:Notificacion-Proceso-Actualizacion-Alarmas'

// Cuenta A
def ACCOUNT_A_ID = '111111111111'
def ACCOUNT_A_NAME = 'Cuenta-A'
def ACCOUNT_A_STAGE_NAME = 'Procesando Cambios en Cuenta A'
def ACCOUNT_A_NOTIFICATION_STAGE_NAME = 'Notificación del proceso Cuenta A'

// Cuenta B
def ACCOUNT_B_ID = '222222222222'
def ACCOUNT_B_NAME = 'Cuenta-B'
def ACCOUNT_B_STAGE_NAME = 'Procesando Cambios en Cuenta B'
def ACCOUNT_B_NOTIFICATION_STAGE_NAME = 'Notificación del proceso Cuenta B'

// Nombre base de las alarmas CloudWatch.
def ALARM_NAME_PREFIX = 'Alarma-Error-Funcion-Lambda'

// Prefijo para el topic SNS creado en cada cuenta.
def SNS_TOPIC_PREFIX = 'Alarma-Error-Lambda-cuenta'

// Sufijo aleatorio para la sesión STS.
def RANDOM_STRING = UUID.randomUUID().toString().substring(0, 7)

// ================================================================
// FUNCIONES DEL PIPELINE
// ================================================================

def processAccount(String accountId, String accountName) {
    withCredentials([[
        $class: 'AmazonWebServicesCredentialsBinding',
        credentialsId: AWS_CREDENTIALS_ID
    ]]) {
        withEnv([
            "AWS_DEFAULT_REGION=${AWS_REGION}",
            "AWS_REGION=${AWS_REGION}",
            "TARGET_ACCOUNT_ID=${accountId}",
            "TARGET_ACCOUNT_NAME=${accountName}",
            "TARGET_ROLE_NAME=${AWS_ROLE_NAME}",
            "NOTIFICATION_EMAIL=${NOTIFICATION_EMAIL}",
            "ALARM_NAME_PREFIX=${ALARM_NAME_PREFIX}",
            "SNS_TOPIC_PREFIX=${SNS_TOPIC_PREFIX}",
            "RANDOM_STRING=${RANDOM_STRING}"
        ]) {
            sh '''#!/bin/bash
                set -e

                echo "============================================================"
                echo "Procesando cuenta: ${TARGET_ACCOUNT_NAME} (${TARGET_ACCOUNT_ID})"
                echo "Región AWS: ${AWS_REGION}"
                echo "============================================================"

                command -v aws >/dev/null 2>&1 || { echo "ERROR: aws-cli no está instalado en el agente Jenkins"; exit 1; }
                command -v jq >/dev/null 2>&1 || { echo "ERROR: jq no está instalado en el agente Jenkins"; exit 1; }

                role_arn="arn:aws:iam::${TARGET_ACCOUNT_ID}:role/${TARGET_ROLE_NAME}"

                echo "Asumiendo rol: ${role_arn}"

                credentials=$(aws sts assume-role \
                    --role-arn "${role_arn}" \
                    --role-session-name "SessionName-${RANDOM_STRING}" \
                    --output json)

                export AWS_ACCESS_KEY_ID=$(echo "${credentials}" | jq -r '.Credentials.AccessKeyId')
                export AWS_SECRET_ACCESS_KEY=$(echo "${credentials}" | jq -r '.Credentials.SecretAccessKey')
                export AWS_SESSION_TOKEN=$(echo "${credentials}" | jq -r '.Credentials.SessionToken')

                echo "Identidad AWS asumida:"
                aws sts get-caller-identity

                sns_topic_name="${SNS_TOPIC_PREFIX}-${TARGET_ACCOUNT_NAME}"
                sns_display_name="Alarma Error Función Lambda en cuenta ${TARGET_ACCOUNT_NAME}"

                echo "Creando o reutilizando SNS Topic: ${sns_topic_name}"
                sns_topic_arn=$(aws sns create-topic \
                    --name "${sns_topic_name}" \
                    --query 'TopicArn' \
                    --output text)

                aws sns set-topic-attributes \
                    --topic-arn "${sns_topic_arn}" \
                    --attribute-name DisplayName \
                    --attribute-value "${sns_display_name}"

                existing_subscription=$(aws sns list-subscriptions-by-topic \
                    --topic-arn "${sns_topic_arn}" \
                    --query "Subscriptions[?Endpoint=='${NOTIFICATION_EMAIL}'].SubscriptionArn" \
                    --output text)

                if [ -z "${existing_subscription}" ]; then
                    echo "Creando suscripción email: ${NOTIFICATION_EMAIL}"
                    aws sns subscribe \
                        --topic-arn "${sns_topic_arn}" \
                        --protocol email \
                        --notification-endpoint "${NOTIFICATION_EMAIL}"

                    echo "IMPORTANTE: confirma la suscripción desde el correo ${NOTIFICATION_EMAIL}"
                else
                    echo "La suscripción email ya existe para ${NOTIFICATION_EMAIL}"
                fi

                echo "Listando funciones Lambda..."
                lambda_functions=$(aws lambda list-functions \
                    --query 'Functions[*].FunctionName' \
                    --output text)

                if [ -z "${lambda_functions}" ]; then
                    echo "No se encontraron funciones Lambda en la cuenta ${TARGET_ACCOUNT_NAME}"
                else
                    for function in ${lambda_functions}; do
                        alarm_name="${ALARM_NAME_PREFIX}-${function}"

                        alarm_exists=$(aws cloudwatch describe-alarms \
                            --alarm-names "${alarm_name}" \
                            --query 'MetricAlarms[0].AlarmName' \
                            --output text 2>/dev/null || true)

                        if [ "${alarm_exists}" = "None" ] || [ -z "${alarm_exists}" ]; then
                            echo "Creando alarma para Lambda: ${function}"
                        else
                            echo "Actualizando alarma existente para Lambda: ${function}"
                        fi

                        aws cloudwatch put-metric-alarm \
                            --alarm-name "${alarm_name}" \
                            --metric-name Errors \
                            --namespace AWS/Lambda \
                            --statistic Sum \
                            --period 300 \
                            --threshold 0 \
                            --comparison-operator GreaterThanThreshold \
                            --dimensions Name=FunctionName,Value="${function}" \
                            --evaluation-periods 1 \
                            --alarm-actions "${sns_topic_arn}"
                    done
                fi

                unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN

                echo "Proceso completado para cuenta: ${TARGET_ACCOUNT_NAME}"
            '''
        }
    }
}

def notifyProcessFinished(String accountName) {
    if (!PROCESS_NOTIFICATION_TOPIC_ARN?.trim()) {
        echo "PROCESS_NOTIFICATION_TOPIC_ARN está vacío. Se omite la notificación de proceso para ${accountName}."
        return
    }

    withCredentials([[
        $class: 'AmazonWebServicesCredentialsBinding',
        credentialsId: AWS_CREDENTIALS_ID
    ]]) {
        withEnv([
            "AWS_DEFAULT_REGION=${AWS_REGION}",
            "AWS_REGION=${AWS_REGION}",
            "PROCESS_NOTIFICATION_TOPIC_ARN=${PROCESS_NOTIFICATION_TOPIC_ARN}",
            "TARGET_ACCOUNT_NAME=${accountName}"
        ]) {
            sh '''#!/bin/bash
                set -e

                aws sns publish \
                    --topic-arn "${PROCESS_NOTIFICATION_TOPIC_ARN}" \
                    --message "Proceso de Actualización de Alarmas para Funciones Lambda en la cuenta ${TARGET_ACCOUNT_NAME} Completado"
            '''
        }
    }
}

// ================================================================
// PIPELINE
// ================================================================

pipeline {
    agent any

    stages {
        stage('Validar configuración') {
            steps {
                script {
                    if (!AWS_CREDENTIALS_ID?.trim()) {
                        error('AWS_CREDENTIALS_ID no puede estar vacío')
                    }
                    if (!AWS_REGION?.trim()) {
                        error('AWS_REGION no puede estar vacío')
                    }
                    if (!AWS_ROLE_NAME?.trim()) {
                        error('AWS_ROLE_NAME no puede estar vacío')
                    }
                    if (!NOTIFICATION_EMAIL?.trim()) {
                        error('NOTIFICATION_EMAIL no puede estar vacío')
                    }
                    if (!ACCOUNT_A_ID?.trim() || !ACCOUNT_A_NAME?.trim()) {
                        error('Debes configurar ACCOUNT_A_ID y ACCOUNT_A_NAME')
                    }
                    if (!ACCOUNT_B_ID?.trim() || !ACCOUNT_B_NAME?.trim()) {
                        error('Debes configurar ACCOUNT_B_ID y ACCOUNT_B_NAME')
                    }

                    echo 'Configuración validada correctamente.'
                }
            }
        }

        stage("${ACCOUNT_A_STAGE_NAME}") {
            steps {
                script {
                    processAccount(ACCOUNT_A_ID, ACCOUNT_A_NAME)
                }
            }
        }

        stage("${ACCOUNT_A_NOTIFICATION_STAGE_NAME}") {
            steps {
                script {
                    notifyProcessFinished(ACCOUNT_A_NAME)
                }
            }
        }

        stage("${ACCOUNT_B_STAGE_NAME}") {
            steps {
                script {
                    processAccount(ACCOUNT_B_ID, ACCOUNT_B_NAME)
                }
            }
        }

        stage("${ACCOUNT_B_NOTIFICATION_STAGE_NAME}") {
            steps {
                script {
                    notifyProcessFinished(ACCOUNT_B_NAME)
                }
            }
        }

        stage('Cerrando sesión') {
            steps {
                sh '''#!/bin/bash
                    unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
                    echo "Credenciales temporales limpiadas."
                '''
            }
        }
    }
}
