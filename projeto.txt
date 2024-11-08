import sqlite3                                                       # Banco de dados 
from kivy.lang import Builder                                        # Interface / Layout
from kivy.uix.screenmanager import ScreenManager, Screen             # Telas diferentes, Ex: Principal e a dos veículos já cadastrados (BD)
from kivymd.app import MDApp                                         # Design material (funcionalidades)
from kivymd.uix.dialog import MDDialog                               # Caixas de diálogo (mensagem de erro, por exemplo)
from kivymd.uix.list import OneLineListItem, OneLineAvatarListItem   # Mostrar itens de uma lista em uma única linha
from kivymd.uix.button import MDRaisedButton, MDFlatButton           # Botões
from kivy.uix.boxlayout import BoxLayout                             # Importa BoxLayout 

class Database:                                                      # Criando a classe do Banco de Dados
    def __init__(self):
        self.connection = sqlite3.connect('vehicles.db')
        self.create_table()

    def create_table(self):                                          # Criando uma tabela nas seguintes condições:
        with self.connection:
            self.connection.execute(''' 
                CREATE TABLE IF NOT EXISTS vehicles (                
                    id INTEGER PRIMARY KEY AUTOINCREMENT,
                    car_name TEXT,
                    model TEXT,
                    plate TEXT,
                    color TEXT,
                    owner_name TEXT
                )
            ''')

    def insert_vehicle(self, car_name, model, plate, color, owner_name):  # Inserir veículos
        with self.connection:
            self.connection.execute(''' 
                INSERT INTO vehicles (car_name, model, plate, color, owner_name)
                VALUES (?, ?, ?, ?, ?) 
            ''', (car_name, model, plate, color, owner_name))

    def fetch_vehicles(self):                     # Consulta o BD
        cursor = self.connection.cursor()
        cursor.execute('SELECT * FROM vehicles')  # Consulta no bd "vehicles"
        return cursor.fetchall()                  # Retorna todos os valores encontrados na lista

    def update_vehicle(self, vehicle_id, car_name, model, plate, color, owner_name):  # Atualiza as informações de um veículo
        with self.connection:
            self.connection.execute(''' 
                UPDATE vehicles
                SET car_name = ?, model = ?, plate = ?, color = ?, owner_name = ?
                WHERE id = ?
            ''', (car_name, model, plate, color, owner_name, vehicle_id))

    def delete_vehicle(self, vehicle_id):  # Exclui o veículo do banco de dados
        with self.connection:
            self.connection.execute(''' 
                DELETE FROM vehicles WHERE id = ?
            ''', (vehicle_id,))

    def close(self):  # Fechar o BD
        self.connection.close()

KV = '''
ScreenManager:
    MainScreen:
    VehicleListScreen:

<MainScreen>:
    name: 'main'
    BoxLayout:
        orientation: 'vertical'
        padding: dp(10)
        spacing: dp(10)

        MDTopAppBar:
            title: "Cadastro de Veículos"
            left_action_items: [["menu", lambda x: app.change_screen('list')]]

        MDTextField:
            id: car_name
            hint_text: "Nome do Carro"
            mode: "rectangle"

        MDTextField:
            id: model
            hint_text: "Modelo"
            mode: "rectangle"

        MDTextField:
            id: plate
            hint_text: "Placa"
            mode: "rectangle"

        MDTextField:
            id: color
            hint_text: "Cor"
            mode: "rectangle"

        MDTextField:
            id: owner_name
            hint_text: "Nome do Proprietário"
            mode: "rectangle"

        MDRaisedButton:
            text: "Adicionar Veículo"
            pos_hint: {"center_x": 0.5}
            on_release: app.add_vehicle()

        MDRaisedButton:
            id: save_button
            text: "Salvar Alterações"
            pos_hint: {"center_x": 0.5}
            on_release: app.save_edited_vehicle()
            disabled: True

<VehicleListScreen>:
    name: 'list'
    BoxLayout:
        orientation: 'vertical'
        padding: dp(10)
        spacing: dp(10)

        MDTopAppBar:
            title: "Lista de Veículos"
            left_action_items: [["arrow-left", lambda x: app.change_screen('main')]]

        ScrollView:
            MDList:
                id: vehicle_list
'''

class MainScreen(Screen):
    pass

class VehicleListScreen(Screen):
    pass

class VehicleApp(MDApp):
    db = Database()            # Inicializa o banco de dados
    editing_vehicle_id = None  # Armazena o id do veículo sendo editado

    def build(self):           #Carrega layout int. gráfica
        return Builder.load_string(KV)

    def change_screen(self, screen_name):
        self.root.current = screen_name
        if screen_name == 'list':
            self.load_vehicles()  # Carrega veículos ao abrir a lista

    def add_vehicle(self):        # Adicionar veiculos
        car_name = self.root.get_screen('main').ids.car_name.text
        model = self.root.get_screen('main').ids.model.text
        plate = self.root.get_screen('main').ids.plate.text
        color = self.root.get_screen('main').ids.color.text
        owner_name = self.root.get_screen('main').ids.owner_name.text

        if all([car_name, model, plate, color, owner_name]):
            self.db.insert_vehicle(car_name, model, plate, color, owner_name)
            self.clear_fields()
            self.load_vehicles()  # Recarrega a lista de veículos
            self.show_success_dialog("Veículo cadastrado com sucesso!")  # Exibe a mensagem de sucesso
        else:
            self.show_error_dialog("Preencha todos os campos corretamente.")

    def load_vehicles(self):  # Carregar veículos
        vehicle_list = self.root.get_screen('list').ids.vehicle_list
        vehicle_list.clear_widgets()
        for vehicle in self.db.fetch_vehicles():
            item = BoxLayout(orientation='horizontal', size_hint_y=None, height="40dp")

            item.add_widget(OneLineListItem(
                text=f"{vehicle[1]} - {vehicle[2]} - {vehicle[3]} - {vehicle[4]} - {vehicle[5]}",
                on_release=lambda item, vehicle=vehicle: self.edit_vehicle(vehicle)  # Chama a função de edição
            ))

            delete_button = MDRaisedButton(
                text="Excluir", 
                size_hint=(None, None), 
                size=("100dp", "40dp"), 
                pos_hint={"center_y": 0.5}, 
                on_release=lambda item, vehicle=vehicle: self.confirm_delete(vehicle)  # Função para confirmar exclusão
            )  # lambda = função sem nome

            item.add_widget(delete_button)  # Adiciona o botão de exclusão ao lado direito do item
            vehicle_list.add_widget(item)

    def edit_vehicle(self, vehicle):  # Carrega os dados do veículo para edição
        self.editing_vehicle_id = vehicle[0]  # Armazena o id do veículo a ser editado
        self.root.get_screen('main').ids.car_name.text = vehicle[1]
        self.root.get_screen('main').ids.model.text = vehicle[2]
        self.root.get_screen('main').ids.plate.text = vehicle[3]
        self.root.get_screen('main').ids.color.text = vehicle[4]
        self.root.get_screen('main').ids.owner_name.text = vehicle[5]
        self.root.get_screen('main').ids.save_button.disabled = False  # Ativa o botão de salvar alterações

    def save_edited_vehicle(self):
        car_name = self.root.get_screen('main').ids.car_name.text
        model = self.root.get_screen('main').ids.model.text
        plate = self.root.get_screen('main').ids.plate.text
        color = self.root.get_screen('main').ids.color.text
        owner_name = self.root.get_screen('main').ids.owner_name.text

        if all([car_name, model, plate, color, owner_name, self.editing_vehicle_id]):
            self.db.update_vehicle(self.editing_vehicle_id, car_name, model, plate, color, owner_name)
            self.clear_fields()
            self.load_vehicles()  # Recarrega a lista de veículos
            self.editing_vehicle_id = None  # Limpa o id de edição
            self.root.get_screen('main').ids.save_button.disabled = True  # Desativa o botão de salvar
            self.show_success_dialog("Alterações salvas!")  # Exibe a mensagem de sucesso
        else:
            self.show_error_dialog("Preencha todos os campos corretamente.")

    def confirm_delete(self, vehicle):  # Confirma a exclusão do veículo
        dialog = MDDialog(
            title="Confirmar Exclusão",
            text=f"Tem certeza de que deseja excluir o veículo {vehicle[1]}?",
            size_hint=(0.8, 1),
            buttons=[
                MDFlatButton(text="Cancelar", on_release=lambda x: dialog.dismiss()),
                MDFlatButton(text="Excluir", on_release=lambda x: self.delete_vehicle(vehicle, dialog))
            ]
        )
        dialog.open()

    def delete_vehicle(self, vehicle, dialog):  # Exclui o veículo do banco de dados
        self.db.delete_vehicle(vehicle[0])
        self.load_vehicles()  # Recarrega a lista de veículos
        dialog.dismiss()  # Fecha o diálogo de confirmação
        self.show_success_dialog("Veículo excluído com sucesso!")  # Exibe a mensagem de sucesso

    def clear_fields(self):
        for field in ['car_name', 'model', 'plate', 'color', 'owner_name']:
            self.root.get_screen('main').ids[field].text = ""

    def show_error_dialog(self, message):
        dialog = MDDialog(title="Erro", text=message, size_hint=(0.8, 1), buttons=[
            MDFlatButton(text="OK", on_release=lambda x: dialog.dismiss())
        ])
        dialog.open()

    
        dialog = MDDialog(title="Sucesso", text=message, size_hint=(0.8, 1), buttons=[
            MDFlatButton(text="OK", on_release=lambda x: dialog.dismiss())
        ])
        dialog.open()

    def on_stop(self):
        self.db.close()  # Fecha o banco de dados ao encerrar o aplicativo


VehicleApp().run()
