# SqlAlchemyNote
## Guida base a l'ORM SqlAlchemy


SqlAlchemy è un ORM per gestire database in python

## controllare la versione installata

```python
>>> import sqlalchemy
>>> sqlalchemy.__version__
```

## Connessione a un db

```python
>>> from sqlalchemy import create_engine
>>> engine = create_engine("sqlite:///:memory:", echo=True)
```

Enginee rappresenta l'interfaccia principale del database e quando viene restituito per la prima volta da create_engine() 
non ha ancora provato a connettersi al database; La connessione vera si instaura quando si richiese eseguire un'attività sul database (Lazy connection).




## Mdellare le relazioni tra entità con SQLAlchemy

1. Creare una tabella

```python
class Persona(Base):
    __tablename__ = 'persone'
    id = Column(Integer, primary_key=True)
    nome = Column(String)
    cognome = Column(String)
    eta = Column(Integer)
    citta_preferite = relationship("Citta", secondary=assoc_persone_citta)
    def __repr__(self):
        return f"<Persona(nome='{self.nome}', cognome='{self.cognome}', eta={self.eta})>"
```

2. creare un oggetto e salvarlo

```python
new_persona = Persona(nome='Mario', cognome='Rossi', eta=35)
session.add(new_persona)
session.commit()
```

3. Recuperare i dati

```python
persone_over_30 = session.query(Persona).filter(Persona.eta > 30)
for persona in persone_over_30:
    print(persona)
```


## Relazioni tra tabelle

Supponiamo di avere due tabelle, una chiamata "students" e l'altra chiamata "courses". 
Gli studenti possono iscriversi a più corsi e ogni corso può essere seguito da molti 
studenti. Questa relazione può essere rappresentata da una relazione molti-a-molti e 
sarà implementata mediante una terza tabella di associazione chiamata "enrollment".


```python
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship, backref
from sqlalchemy.ext.declarative import declarative_base

engine = create_engine('sqlite:///example.db')
Base = declarative_base()

class Student(Base):
    __tablename__ = 'students'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    
    enrollments = relationship("Enrollment", backref="student")

class Course(Base):
    __tablename__ = 'courses'
    id = Column(Integer, primary_key=True)
    title = Column(String)
    
    enrollments = relationship("Enrollment", backref="course")

class Enrollment(Base):
    __tablename__ = 'enrollments'
    id = Column(Integer, primary_key=True)
    student_id = Column(Integer, ForeignKey('students.id'))
    course_id = Column(Integer, ForeignKey('courses.id'))
```

In questo esempio, abbiamo introdotto le cardinalità. La relazione tra gli studenti e 
gli iscriversi (enrollments) è definita come "one-to-many". In altre parole, ogni 
studente può avere molteplici "enrollments", ma ciascuna iscrizione appartiene 
solo a uno studente. Questo è indicato dalla proprietà "enrollments" nella classe 
Student, che è esplicitamente una relazione "one-to-many" con la classe Enrollment.

La relazione tra i corsi e gli iscritti è definita in modo simile. Tuttavia, 
dal lato del corso, l'associazione è "one-to-many" in cui ogni corso può avere molteplici "enrollments".

Infine, la relazione tra le tabella degli studenti e dei corsi e quella degli 
iscritti è definita come "many-to-many". Ciò significa che ogni studente può 
essere iscritto a molti corsi e ogni corso può essere seguito da molti studenti. 
La cardinalità è rappresentata dalla classe Enrollment che ha una relazione 
"many-to-one" con Student e con Course.


## Back_populates, ForeignKey, relationship

1. ForeignKey definisce un campo nella tabella figlia che fa riferimento alla chiave primaria della 
tabella padre, mentre relationship definisce la relazione tra le due tabelle.

Ad esempio, supponiamo di avere due tabelle, Book e Author, dove 
Book ha un campo author_id che fa riferimento alla chiave primaria 
di Author. La relazione tra le due tabelle può essere definita nel seguente modo:

```python
class Author(Base):
    __tablename__ = 'authors'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    
class Book(Base):
    __tablename__ = 'books'
    id = Column(Integer, primary_key=True)
    title = Column(String)
    author_id = Column(Integer, ForeignKey('authors.id'))
    author = relationship(Author)
```


2. Con back_populates, è possibile definire la relazione in modo bilaterale, 
in modo che quando si accede all'attributo author in Book, sqlalchemy 
automaticamente ottenga l'elenco di libri associati all'autore corrispondente. 


```python
class Author(Base):
    __tablename__ = 'authors'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    books = relationship("Book", back_populates="author")

class Book(Base):
    __tablename__ = 'books'
    id = Column(Integer, primary_key=True)
    title = Column(String)
    author_id = Column(Integer, ForeignKey('authors.id'))
    author = relationship("Author", back_populates="books")
```

Qui, abbiamo aggiunto alla definizione Author la proprieta' books che rappresenta la 
relazione tra le due tabelle books e authors. Mentre nella definizione della 
relazione in Book abbiamo definito back_populates="books" per dichiarare che 
la proprietà author in Book viene mappata dalla proprietà books in Author.

Questo ci consente di accedere facilmente all'elenco dei libri associati a 
un autore attraverso l'attributo books in Author:

```python
author = session.query(Author).first()
print(author.books)
```

## Esempio - Autenticazione Mutiruolo

La prima tabella, Utente, contiene informazioni sugli utenti registrati nel sistema, compreso il loro nome utente, la password e la data di creazione 
dell'account. Ogni utente può essere associato a zero o a un unico profilo, che rappresenta l'insieme di impostazi per l'utente

Ogni profilo è associato a uno o più settori, che possono essere utilizzati per impostare restrizioni di accesso. 

La tabella Settore contiene informazioni sui settori a cui l'utente ha accesso. Ogni settore è associato ad un profilo e può contenere uno o più ruoli, 
che rappresentano i permessi specifici all'interno del settore. 

La tabella Ruolo definisce i permessi degli utenti all'interno dei vari settori. Ogni ruolo è definito come l'insieme di endpoit o rotte accessibili a l'utente

La tabella Rotta contiene l'elenco delle rotte (endpoint rest) disponibili per l'applicazione

E' possibile creare un utente senza assegnargli un profilo. La relazione tra l'entità "Utente" e "Profilo" è 
definita come uno-a-uno opzionale, quindi l'utente può essere creato anche senza un profilo associato. Inoltre, l'attributo 
"profilo" nella classe "Utente" è impostato su "uselist=False", il che significa che la relazione è singolare e non obbligatoria. 


```python
class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True, autoincrement=True, index=True)
    username = Column(String(50), unique=True, nullable=False)
    name = Column(String(50), nullable=False)
    cognome = Column(String(50), nullable=False)
    CF = Column(String(16), unique=True, nullable=True)
    password = Column(String(100), nullable=False)
    email = Column(String(100), unique=True, nullable=False)
    superuser = Column(Boolean, unique=False, default=False)
    creator = Column(String(50), nullable=False)
    id_creator = Column(Integer, nullable=False)
    creation_date = Column(String(10), nullable=False)
    active = Column(Boolean, unique=False, default=False)

    """
    profile = relationship("Profilo", uselist=False, backref="user")

    La keyword "uselist" specifica se la relazione è multipla o singola. 
    Impostata a False indica che la relazione è singolare, cioè che
    un utente può avere solo un profilo associato ad esso.

    La keyword "backref" definisce una relazione inversa tra le tabelle. Q
    ui sta definendo una relazione inversa da "Profilo" a "Utente" utilizzando il nome "utente".
    Quindi ogni profilo avrà un attributo "utente" che farà riferimento all'utente associato ad esso.

    Quindi con quella riga si definisce una relazione uno-a-uno tra l'entità "Utente" e l'entità "Profilo", quindi
    ogni utente può avere solo un profilo associato ad esso e ogni profilo avrà un attributo "utente" che farà 
    riferimento all'utente associato ad esso.
    """
    profile = relationship("Profilo", uselist=False, backref="user")

    def as_dict(self):
        return {c.name: getattr(self, c.name) for c in self.__table__.columns}


class Profile(Base):
    __tablename__ = "profiles"
    id = Column(Integer, primary_key=True)
    name = Column(String(50), nullable=False)
    description = Column(String(100))
    active = Column(Boolean, nullable=False)
    only_observer = Column(Boolean, nullable=True)
    defaultSector = Column(Integer, nullable=False) # settore di default
    # lista settori a cui si è abilitati come osservatori - ARRAY specifica un array di valori
    observed_sectors = Column(ARRAY(Integer), nullable=False) 
    user_id = Column(Integer, ForeignKey("users.id"))

    """
    La riga: sectors = relationship("Sector", backref="profile")

    definisce una relazione "one-to-many" tra le classi "Profilo" e "Settore".

    La relazione "relationship" ha due opzioni: 
    il nome della classe di destinazione della relazione ("Settore") e l'opione "backref" che fa riferimento
    al nome dell'attributo della classe di destinazione che permette di accedere alla classe di origine ("Profilo").

    quindi permette

    - Per ogni istanza di "Profilo", permette di accedere a una lista di istanze di "Settore" con il metodo "profilo.settori" (generato automaticamente dal "backref").
    - Per ogni istanza di "Settore", permette di accedere all'istanza di "Profilo" che lo contiene con il metodo "settore.profilo".
    """
    sectors = relationship("Sector", backref="profile")

    def as_dict(self):
        return {c.name: getattr(self, c.name) for c in self.__table__.columns}


class Sector(Base):
    __tablename__ = "sectors"
    id = Column(Integer, primary_key=True)
    name = Column(String(50), nullable=False)
    description = Column(String(100))
    reserved = Column(Boolean, nullable=False)
    profile_id = Column(Integer, ForeignKey("profiles.id"))
    roles = relationship("Role", backref="sector")

    def as_dict(self):
        return {c.name: getattr(self, c.name) for c in self.__table__.columns}


class Role(Base):
    __tablename__ = "roles"
    id = Column(Integer, primary_key=True)
    name = Column(String(50), nullable=False)
    description = Column(String(100), nullable=False)
    sector_id = Column(Integer, ForeignKey("sectors.id"))
    
    """
    La riga:  routes = relationship("Route", secondary="roles_routes")

    crea una relazione "MOLTI A MOLTI" tra le tabelle "roles" e "routes" tramite la tabella 
    di relazione "roles_routes".
    
    Secondary indica la tavola di associazione che viene utilizzata per 
    associare le due tabelle "Route" e "roles_routes". Quindi indica la tabella 
    che rappresenta la relazione molti-a-molti tra le tabelle "Route" e "roles_routes".
    """
    
    routes = relationship("Route", secondary="roles_routes")

    def as_dict(self):
        return {c.name: getattr(self, c.name) for c in self.__table__.columns}

class Route(Base):
    __tablename__ = "routes"
    id = Column(Integer, primary_key=True)
    name = Column(String(50), nullable=False)
    path = Column(String(50), nullable=False)
    description = Column(String(100), nullable=False)
    active = Column(Boolean, nullable=False)
    
    def as_dict(self):
        return {c.name: getattr(self, c.name) for c in self.__table__.columns}

"""
La tabella "roles_routes" è una tabella di relazione e viene utilizzata 
come una tabella di collegamento tra le tabelle "roles" e "routes" per creare 
una relazione MOLTI A MOLTI tra le due tabelle.
"""
roles_routes = Table("roles_routes", Base.metadata,
                    Column("role_id", Integer, ForeignKey("roles.id")),
                    Column("route_id", Integer, ForeignKey("routes.id"))
                    )

```


## Controllare se tutte le tabelle sono disponibili

```python
def check_tables():
    from sqlalchemy import inspect
    insp = inspect(engine)
    table_names = insp.get_table_names()
    required_tables = ['users','profiles', 'routes', 'roles', 'roles_routes']
    return all(table in table_names for table in required_tables)
```
