import logging
from telegram import Update
from telegram.ext import Application, MessageHandler, filters, CallbackContext
import requests
from apscheduler.schedulers.asyncio import AsyncIOScheduler
from pytz import timezone
import nest_asyncio
from bs4 import BeautifulSoup
import os
from dotenv import load_dotenv

# Cargar las variables de entorno desde el archivo .env
load_dotenv()

# Verificación de que las variables de entorno se cargaron correctamente
print("API_FOOTBALL_KEY:", os.getenv("API_FOOTBALL_KEY"))
print("API_FOOTBALL_DATA_KEY:", os.getenv("API_FOOTBALL_DATA_KEY"))
print("BOT_TOKEN:", os.getenv("BOT_TOKEN"))

# Aplica la corrección para manejar el loop en entornos ya existentes
nest_asyncio.apply()

# Inicia el scheduler sin pasar la zona horaria explícitamente
scheduler = AsyncIOScheduler(timezone=None)

# Configura el logging
logging.basicConfig(format='%(asctime)s - %(name)s - %(levelname)s - %(message)s', level=logging.INFO)
logger = logging.getLogger(__name__)

# APIs que vamos a usar
API_FOOTBALL_URL = "https://v3.football.api-sports.io/"
API_FOOTBALL_KEY = os.getenv("API_FOOTBALL_KEY")  # Obtener la clave de API desde las variables de entorno
API_FOOTBALL_HEADERS = {"x-apisports-key": API_FOOTBALL_KEY}

API_FOOTBALL_DATA_URL = "https://api.football-data.org/v2/"  # Otra API
API_FOOTBALL_DATA_KEY = os.getenv("API_FOOTBALL_DATA_KEY")  # Obtener la clave de API desde las variables de entorno
API_FOOTBALL_DATA_HEADERS = {"X-Auth-Token": API_FOOTBALL_DATA_KEY}

# Función para obtener bajas de Transfermarkt
def get_injuries_transfermarkt(team_name):
    """Obtiene las bajas de un equipo de Transfermarkt."""
    url = f"https://www.transfermarkt.com/{team_name}/verletzungen/verein/alle"
    
    try:
        response = requests.get(url)
        response.raise_for_status()  # Lanzará un error si el status code no es 200
        soup = BeautifulSoup(response.text, 'html.parser')
        
        # Buscamos la sección donde se encuentran las bajas
        injuries_table = soup.find('table', {'class': 'items'})
        
        if not injuries_table:
            return 0  # Si no se encuentra la tabla de bajas, devolvemos 0
        
        injuries = injuries_table.find_all('tr')
        
        # Filtramos las bajas de jugadores
        return len([injury for injury in injuries if 'Verletzung' in injury.text])
    
    except requests.exceptions.RequestException as e:
        print(f"Error de conexión al obtener las bajas de Transfermarkt: {e}")
        return 0

# Función para obtener bajas de Flashscore
def get_injuries_flashscore(team_name):
    """Obtiene las bajas de un equipo de Flashscore."""
    url = f"https://www.flashscore.com/futbol/{team_name}/bajas/"
    
    try:
        response = requests.get(url)
        response.raise_for_status()  # Lanzará un error si el status code no es 200
        soup = BeautifulSoup(response.text, 'html.parser')
        
        # Buscar las bajas en la página de Flashscore
        injuries_section = soup.find('section', {'class': 'injuries-section'})
        
        if not injuries_section:
            return 0  # Si no hay información de bajas, devolvemos 0
        
        injuries = injuries_section.find_all('div', {'class': 'injury-player'})
        
        return len(injuries)
    
    except requests.exceptions.RequestException as e:
        print(f"Error de conexión al obtener las bajas de Flashscore: {e}")
        return 0

# Función para obtener las bajas desde API-FOOTBALL
def get_injuries_api(team_id):
    """Obtiene las bajas de un equipo desde la API-FOOTBALL."""
    try:
        injuries_response = requests.get(f"{API_FOOTBALL_URL}players/sidelined", headers=API_FOOTBALL_HEADERS, params={"team": team_id})
        injuries_response.raise_for_status()  # Lanzará un error si el status code no es 200
        injuries_data = injuries_response.json()
        
        return len(injuries_data.get('response', []))
    
    except requests.exceptions.RequestException as e:
        print(f"Error al obtener las bajas desde la API de Football: {e}")
        return 0

async def get_match_prediction(team1, team2):
    """Obtiene una predicción basada en estadísticas recientes, factor campo y bajas importantes."""
    try:
        # Buscar ID de los equipos usando API-FOOTBALL
        response1 = requests.get(API_FOOTBALL_URL + "teams", headers=API_FOOTBALL_HEADERS, params={"search": team1})
        response2 = requests.get(API_FOOTBALL_URL + "teams", headers=API_FOOTBALL_HEADERS, params={"search": team2})

        response1.raise_for_status()  # Lanzará un error si el status code no es 200
        response2.raise_for_status()  # Lanzará un error si el status code no es 200

        try:
            team1_data = response1.json()['response']
            team2_data = response2.json()['response']
            if not team1_data or not team2_data:
                return f"Uno o ambos equipos '{team1}' y '{team2}' no fueron encontrados. Verifica los nombres."
            
            team1_id = team1_data[0]['team']['id']
            team2_id = team2_data[0]['team']['id']
        except (KeyError, IndexError):
            return f"Hubo un error al procesar los equipos '{team1}' y '{team2}'."

        # Obtener enfrentamientos previos con API-FOOTBALL
        matches = requests.get(API_FOOTBALL_URL + "fixtures/headtohead", headers=API_FOOTBALL_HEADERS, params={"h2h": f"{team1_id}-{team2_id}"})
        matches.raise_for_status()  # Lanzará un error si el status code no es 200
        data = matches.json().get('response', [])

        if not data:
            return f"No se encontraron enfrentamientos previos entre {team1} y {team2}. 😕"

        wins_team1 = sum(1 for match in data if match['teams']['home']['id'] == team1_id and match['teams']['home']['winner'])
        wins_team2 = sum(1 for match in data if match['teams']['away']['id'] == team2_id and match['teams']['away']['winner'])
        draws = len(data) - (wins_team1 + wins_team2)
        
        # Obtener bajas importantes con la otra API (Football-Data.org)
        injuries1 = requests.get(f"{API_FOOTBALL_DATA_URL}teams/{team1_id}/injuries", headers=API_FOOTBALL_DATA_HEADERS)
        injuries2 = requests.get(f"{API_FOOTBALL_DATA_URL}teams/{team2_id}/injuries", headers=API_FOOTBALL_DATA_HEADERS)

        injuries1.raise_for_status()  # Lanzará un error si el status code no es 200
        injuries2.raise_for_status()  # Lanzará un error si el status code no es 200

        key_absences1 = len(injuries1.json().get('injuries', []))
        key_absences2 = len(injuries2.json().get('injuries', []))

        # Obtener bajas de Transfermarkt y Flashscore
        key_absences1 += get_injuries_transfermarkt(team1)
        key_absences2 += get_injuries_transfermarkt(team2)
        key_absences1 += get_injuries_flashscore(team1)
        key_absences2 += get_injuries_flashscore(team2)

        # Verificación de las bajas
        absences_message1 = f"Bajas importantes: {key_absences1}" if key_absences1 > 0 else "No hay bajas importantes para este equipo."
        absences_message2 = f"Bajas importantes: {key_absences2}" if key_absences2 > 0 else "No hay bajas importantes para este equipo."

        if wins_team1 > wins_team2:
            prediction = f"{team1} tiene más probabilidades de ganar! 🏆"
        elif wins_team2 > wins_team1:
            prediction = f"{team2} parece favorito para este partido! ⚽"
        else:
            prediction = "Parece un partido muy igualado, ¡cualquier cosa puede pasar! 🤔"
        
        # Calcular posibilidad de sorpresa
        surprise_factor = "Baja" if abs(wins_team1 - wins_team2) > 3 else "Media" if abs(wins_team1 - wins_team2) > 1 else "Alta"
        
        # Retornar resultado
        return (f"Historial de enfrentamientos:\n"
                 f"{team1} 🆚 {team2}\n"
                 f"- {team1}: {wins_team1} victorias\n"
                 f"- {team2}: {wins_team2} victorias\n"
                 f"- Empates: {draws}\n\n"
                 f"{prediction}\n"
                 f"📌 {team1}: {absences_message1}\n"
                 f"📌 {team2}: {absences_message2}\n"
                 f"⚠️ Posibilidad de sorpresa: {surprise_factor}")
    except requests.exceptions.RequestException as e:
        return f"Error al obtener los datos del partido: {e}"

async def handle_message(update: Update, context: CallbackContext) -> None:
    """Maneja los mensajes que comienzan con 'Quinibot'.""" 
    text = update.message.text
    if text.lower().startswith("quinibot"):
        message_parts = text.split(" ", 1)
        if len(message_parts) > 1:
            teams = message_parts[1].split(" vs ")
            if len(teams) == 2:
                response = await get_match_prediction(teams[0], teams[1])
            else:
                response = "Formato incorrecto. Pregunta como: 'Quinibot Elche vs Eldense'."
        else:
            response = "¡Hola! Pregúntame sobre partidos de fútbol escribiendo: 'Quinibot Equipo1 vs Equipo2'. ⚽"
        await update.message.reply_text(response)

async def main():
    """Inicia el bot."""    
    # Usar la clave de bot desde las variables de entorno
    application = Application.builder().token(os.getenv("BOT_TOKEN")).build()
    
    # Agregar el handler para mensajes
    application.add_handler(MessageHandler(filters.TEXT & ~filters.COMMAND, handle_message))
    
    # Iniciar el bot
    await application.run_polling()

if __name__ == '__main__':
    import asyncio
    asyncio.run(main())  # Usamos asyncio.run() para ejecutar el bot de forma adecuada.
