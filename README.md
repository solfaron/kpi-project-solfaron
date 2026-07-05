# Базова інформація про проєкт
Назва: `Void Delver`  
Виконавці: `Солопов Данило та Рибаков Ігор.`

Цей проєкт являє собою телеграм бота з [rougelike](https://uk.wikipedia.org/wiki/Roguelike) грою в якій ви за шукача пригод відправляєтеся в глибоке підземелля, яке являє собою величезний замок, щоб дістати артефакт який у вас замовив містичний чоловік із таверни.

Ваша задача пройти через підземелля та витягти звідти той артефакт, але це не буде так просто з двох причин: 
```
1. Це багаторівневе підземелля із смертельними ворагами всередині  
2. За вами постійно женется невідома істота
``` 

Чи зможете ви повернутися залежить тільки від вас, вдачі та обізнаності на підземеллі.


# Структура гри
У грі буде 3 локації:
```
1. Найвища башня замку [🏰]
2. Тронна зала [👑]
3. Підземна в'язниця [💀]
```

На кожній локації буде по три рівні та три вороги:
```
На першій локації - маги
На другій локації - королівські вартові
На третійц - в'язні які перетоврилися на зомбі та скелетів
```


Гравець може посилюватися, підіймаючи рівень після боїв або знаходячи предмети на локаціях. 
У грі присутні 16 предметів.

Після проходження третьої локації гравець потрапляє на арену з фінальним босом.

На кожному рівні локації у гравця буде обмежена кількість ходів (25) через те, що за гравцем постійно женеться звір.  
Якщо ходи вичерпані, гра завершується програшем гравця.  
Після виходу з локації рахунок ходів скидається.

# Інтсаляція та Запуск
Для інсталяції застосунку потрібно:
1. Завантажити найновіший реліз у вкладці Releases на головній сторінці GitHub.
2. Після цього потрібно розпакувати завантажений Release.zip.
3. У директорію із виконувальним файлом потрібно створити файл Token.txt та записати в нього токен свого Телеграм-бота.

# Графічне уявлення взаємодії користувача із ботом
```mermaid
graph TD
    Start([/start]) --> CheckDB[Запрос в БД]
    CheckDB --> Exist{Чи є гравець?}

    Exist -- Ні --> CreateProfile[Створити запис з обраним ніком]
    CreateProfile --> ClassMenu[Меню вибору класу]

    Exist -- Так --> State{Стан?}
    State -- Мертвий/Новий --> ClassMenu
    State -- У грі --> RoomDisplay[Показати теперішню кімнату]

    ClassMenu -->|Натиснув кнопку класу| AssignStats[Задання дефолтних статів]
    AssignStats --> SetInGame[State = InGame]
    SetInGame --> RoomDisplay

    RoomDisplay --> Fight{Бій: HP <= 0?}

    Fight -- Так --> SetDead[State = Dead]
    SetDead ----> ClassMenu

    Fight -- Ні --> NextAction[Наступна дія / кімната]
    NextAction --> RoomDisplay
```
# Архітектура проекта
```mermaid
graph TD
    subgraph App Layer
        B[BotManager : Telegram Bot API]
    end

    subgraph Infrastructure Layer
        S[Services : BattleRuler, MapRuler, GameRuler]
        R[Repositories : SQLite DB]
        F[Factories : EnemyFactory, CharacterFactory]
    end

    subgraph Core Layer
        M[Models : Character, Enemies, Items]
        I[Interfaces : IBattleUnit, ISkill]
        E[Enums : RoomType, BattleStateEnum]
    end

    B -->|Calls| S
    B -->|Fetches Data| R
    S -->|Reads/Writes| R
    S -->|Uses| F
    S -->|Manipulates| M
    F -->|Creates| M
    M -->|Implements| I
```

# Діаграмма класів
```mermaid
classDiagram
    class IBattleUnit {
        <<interface>>
        +int Hp
        +int MaxHp
        +int HandDmg
        +int PhisDefense
        +List~ISkill~ Skills
        +List~ActiveEffect~ CurrentEffects
    }

    class Character {
        +int Location
        +int Floor
        +int TurnsLeft
        +int State
    }

    class EnemyBase {
        +string Name
        +string ClassType
    }

    class ISkill {
        <<interface>>
        +string Name
        +Execute(IBattleUnit attacker, IBattleUnit target)
    }

    class BaseItem {
        <<abstract>>
        +string Name
        +Rarity Rarity
        +AddBonuses(Character hero)
    }

    IBattleUnit <|.. Character : Implements
    IBattleUnit <|.. EnemyBase : Implements
    IBattleUnit "1" *-- "many" ISkill : Has
    BaseItem ..> Character : Modifies
```

# Діаграма послідовності
```mermaid
sequenceDiagram
    actor Player
    participant Bot as BotManager
    participant Ruler as BattleRuler
    participant CharRepo as CharacterRepository
    participant BattleRepo as ActiveBattleRepository
    
    Player->>Bot: Clicks "Attack Target"
    Bot->>Ruler: ProcessAttack(telegramId, enemyIndex)
    
    Ruler->>CharRepo: GetActiveCharacter()
    CharRepo-->>Ruler: Character Object
    
    Ruler->>BattleRepo: GetBattle()
    BattleRepo-->>Ruler: JSON Battle Data
    
    Ruler->>Ruler: Calculate Hero Damage
    
    alt Enemies Alive
        Ruler->>Ruler: Enemy Counter-Attack (Random Skill)
        Ruler->>Ruler: Tick DoT Effects (Poison, Burn)
        Ruler->>BattleRepo: SaveBattleState()
        Ruler-->>Bot: Return Ongoing Status & Msg
    else All Enemies Dead
        Ruler->>CharRepo: UpdateCharacterState(0)
        Ruler->>BattleRepo: EndBattle()
        Ruler-->>Bot: Return Victory Status & Msg
    end
    
    Bot-->>Player: Update Telegram UI
```
:shipit:
