from typing import List, Optional
from sqlmodel import Field, Relationship, Session, SQLModel, create_engine, select
from nicegui import ui, app

# -----------------------------------------------------------------------------
# 1. DATABASE MODELS (The ER Diagram implemented in code)
# -----------------------------------------------------------------------------
class User(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    username: str
    role: str = Field(default="User")  # User, Power User, Admin
    
    proposed_events: List["Event"] = Relationship(back_populates="creator")
    claimed_items: List["FoodItem"] = Relationship(back_populates="claimed_by")

class Event(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    title: str
    description: str
    location: str
    status: str = Field(default="Published") # Pending or Published
    creator_id: int = Field(foreign_key="user.id")
    
    creator: User = Relationship(back_populates="proposed_events")
    food_items: List["FoodItem"] = Relationship(back_populates="event")

class FoodItem(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    item_name: str
    is_claimed: bool = Field(default=False)
    event_id: int = Field(foreign_key="event.id")
    claimed_by_id: Optional[int] = Field(default=None, foreign_key="user.id")
    
    event: Event = Relationship(back_populates="food_items")
    claimed_by: Optional[User] = Relationship(back_populates="claimed_items")

# Setup SQLite Database
engine = create_engine("sqlite:///studento.db")

def init_db():
    SQLModel.metadata.create_all(engine)
    with Session(engine) as session:
        if not session.exec(select(User)).first():
            # Initial Data
            u1 = User(username="Student_User", role="User")
            u2 = User(username="FS_Member", role="Power User")
            session.add_all([u1, u2])
            session.commit()
            # Initial Event
            e1 = Event(title="HSW Networking Apéro", description="Connect with peers!", location="HSW Hall", creator_id=u1.id)
            session.add(e1)
            session.commit()
            session.add_all([FoodItem(item_name="Wine", event_id=e1.id), FoodItem(item_name="Chips", event_id=e1.id)])
            session.commit()

# -----------------------------------------------------------------------------
# 2. UI LOGIC (Event Coordination Page)
# -----------------------------------------------------------------------------

# Helper to get current logged-in user (Defaulting to user ID 1 for this demo)
def get_user_id():
    return app.storage.user.get('user_id', 1)

@ui.page('/')
def index():
    # Navbar
    with ui.header().classes('bg-blue-900 items-center justify-between'):
        ui.label('🎓 Studento').classes('text-2xl font-bold')
        with ui.row():
            ui.button('Home', on_click=lambda: ui.navigate.to('/')).props('flat color=white')
            ui.button('Propose Event', on_click=lambda: ui.navigate.to('/propose')).props('flat color=white')

    with ui.column().classes('w-full max-w-4xl mx-auto p-4'):
        ui.label('Event Plan').classes('text-4xl font-bold text-gray-800 mb-4')
        
        with Session(engine) as session:
            events = session.exec(select(Event).where(Event.status == "Published")).all()
            user = session.get(User, get_user_id())

            for event in events:
                with ui.card().classes('w-full mb-6 p-6 shadow-lg'):
                    with ui.row().classes('w-full justify-between'):
                        with ui.column():
                            ui.label(event.title).classes('text-2xl font-bold text-blue-800')
                            ui.label(f"📍 {event.location}").classes('text-gray-500 italic')
                        ui.badge(event.status, color='green')

                    ui.label(event.description).classes('mt-4 text-lg')
                    
                    # Food List Management (Section PG 3)
                    ui.separator().classes('my-4')
                    ui.label('Food List Coordination').classes('text-xl font-semibold')
                    
                    for item in event.food_items:
                        with ui.row().classes('w-full items-center justify-between py-2 border-b'):
                            ui.label(item.item_name).classes('text-md')
                            
                            if item.is_claimed:
                                ui.label(f"Taken by {item.claimed_by.username}").classes('text-green-600 font-medium')
                            else:
                                ui.button('I will bring this!', on_click=lambda i=item: claim(i.id)).props('rounded color=blue')

def claim(item_id):
    with Session(engine) as session:
        item = session.get(FoodItem, item_id)
        item.is_claimed = True
        item.claimed_by_id = get_user_id()
        session.add(item)
        session.commit()
    ui.notify(f"Thank you for contributing!")
    ui.navigate.reload()

@ui.page('/propose')
def propose_page():
    with ui.header().classes('bg-blue-900'):
        ui.label('🎓 Studento - Propose Event').classes('text-2xl font-bold')
    
    with ui.column().classes('w-full max-w-2xl mx-auto p-8'):
        ui.label('Submit Event Proposal').classes('text-3xl font-bold mb-6')
        
        with ui.card().classes('w-full p-6'):
            t = ui.input('Event Title').classes('w-full mb-2')
            l = ui.input('Location').classes('w-full mb-2')
            d = ui.textarea('Description').classes('w-full mb-4')
            f = ui.input('Food items (separate with commas)').classes('w-full mb-6')

            def submit():
                with Session(engine) as session:
                    new_event = Event(title=t.value, location=l.value, description=d.value, creator_id=get_user_id())
                    session.add(new_event)
                    session.commit()
                    
                    if f.value:
                        for food_name in f.value.split(','):
                            session.add(FoodItem(item_name=food_name.strip(), event_id=new_event.id))
                        session.commit()
                
                ui.notify('Proposal submitted!')
                ui.navigate.to('/')

            ui.button('Submit Proposal', on_click=submit).classes('w-full py-2 bg-blue-800 text-white')

# -----------------------------------------------------------------------------
# 3. START APP
# -----------------------------------------------------------------------------
if __name__ in {"__main__", "__mp_main__"}:
    init_db()
    ui.run(title="Studento Coordination", storage_secret="fhnw_secret")
