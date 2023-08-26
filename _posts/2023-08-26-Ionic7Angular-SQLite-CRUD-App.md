
# Ionic 7 SQLite Database CRUD App Example Tutorial using Angular and @capacitor-community/sqlite
---
last updated on August 26, 2023 by Quéau Jean Pierre

In that tutorial we will learned how to create a Ionic7/Angular basic CRUD application and implement the @capacitor-community/sqlite plugin to store the data in a SQLite database.

The first part of the tutorial will concentrate on how to create that application and run it on a Web browser where the data will be stored on an Indexed database using sql.js and localForage modules.

Thanks to the Ionic Team and their hard work to bring CAPACITOR 5, the second part will concentrate on native platforms (iOS and Android) and also on Electron platform.

## Part 1 - Web - Table of Contents

 - [Install New Ionic Application](#install-new-ionic-application)
 - [Install Required Packages](#install-required-packages)
 - [Create the SQLite Database](#create-the-sqlite-database)
 - [Create CRUD and SQLite Services](#create-crud-and-sqlite-services)
 - [Angular Plugin Implementation](#angular-plugin-implementation)
 - [Users Component Implementation](#users-component-implementation)
 - [Update Config and Package.json Scripts](#update-config-and-packagejson-scripts)
 - [Run the Web SQLite App](#run-the-web-sqlite-app)
 - [Part 1 Conclusion](#part-1-conclusion)

### Install New Ionic Application

 - Install the latest version of Ionic CLI and Cordova globally installed on your device, run the below command.

    ```bash
    sudo npm install -g @ionic/cli
    ```

 - Create a new blank Ionic Angular app

    !!! Select standalone when asked !!!

    ```bash
    ionic start ionic7-angular-sqlite-app blank --type=angular --capacitor
    ```

 - Go inside the project folder

    ```bash
    cd ./ionic7-angular-sqlite-app
    ```

### Install Required Packages

 - run the below commands

    ```bash
    npm install --save @capacitor-community/sqlite
    npm install --save @capacitor/toast
    npm install --save @ionic/pwa-elements
    npm install --save-dev copyfiles
    ```

### Create the SQLite Database

 - To easily upgrade the database from one version to the next, the database is created through an upgrade statement.

    ```bash
    ng g class upgrades/user.upgrade.statements 
    ```

 - Define the upgrade statements in `upgrades/user.upgrade.statements.ts` file

    ```ts
    export class UserUpgradeStatements {
        userUpgrades = [
            {
            toVersion: 1,
            statements: [
                `CREATE TABLE IF NOT EXISTS users(
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                active INTEGER DEFAULT 1
                );`
            ]
            },
            /* add new statements below for next database version when required*/
            /*
            {
            toVersion: 2,
            statements: [
                `ALTER TABLE users ADD COLUMN email TEXT;`,
            ]
            },
            */
        ]
    }    
    ```

 - Define the User Model

    ```bash
    ng g interface models/user
    ```

 - update the `models/user.ts` file as

    ```ts
    export interface User {
        id: number
        name: string
        active: number
        /* for version 2
        email: string
        */
    }
    ```

### Create CRUD and SQLite Services

 - Run the following commands to create CRUD and SQLite services.

    ```bash
    ng g class services/sqlite.service
    ng g class services/storage.service
    ```

 - Open the `services/sqlite.service.ts` file and replace with the following code.

    ```ts
        import { Injectable } from '@angular/core';

        import { Capacitor } from '@capacitor/core';
        import { CapacitorSQLite, SQLiteConnection, SQLiteDBConnection, CapacitorSQLitePlugin,
        capSQLiteUpgradeOptions, capSQLiteResult, capSQLiteValues} from '@capacitor-community/sqlite';

        @Injectable()

        export class SQLiteService {
            sqliteConnection!: SQLiteConnection;
            isService: boolean = false;
            platform!: string;
            sqlitePlugin!: CapacitorSQLitePlugin;
            native: boolean = false;
            constructor() {
            }
            /**
             * Plugin Initialization
             */
            async initializePlugin(): Promise<boolean> {
                    this.platform = Capacitor.getPlatform();
                    if(this.platform === 'ios' || this.platform === 'android') this.native = true;
                    this.sqlitePlugin = CapacitorSQLite;
                    this.sqliteConnection = new SQLiteConnection(this.sqlitePlugin);
                    this.isService = true;
                    return true;
            }

            async initWebStore(): Promise<void> {
                try {
                await this.sqliteConnection.initWebStore();
                } catch(err: any) {
                const msg = err.message ? err.message : err;
                return Promise.reject(`initWebStore: ${err}`);
                }
            }

            async openDatabase(dbName:string, encrypted: boolean, mode: string, version: number, readonly: boolean): Promise<SQLiteDBConnection> {
                let db: SQLiteDBConnection;
                const retCC = (await this.sqliteConnection.checkConnectionsConsistency()).result;
                let isConn = (await this.sqliteConnection.isConnection(dbName, readonly)).result;
                if(retCC && isConn) {
                    db = await this.sqliteConnection.retrieveConnection(dbName, readonly);
                } else {
                    db = await this.sqliteConnection
                            .createConnection(dbName, encrypted, mode, version, readonly);
                }
                await db.open();
                return db;
            }
            async retrieveConnection(dbName:string, readonly: boolean): Promise<SQLiteDBConnection> {
                return await this.sqliteConnection.retrieveConnection(dbName, readonly);
            }
            async closeConnection(database:string, readonly?: boolean): Promise<void> {
                const readOnly = readonly ? readonly : false;
                return await this.sqliteConnection.closeConnection(database, readOnly);
            }
            async addUpgradeStatement(options:capSQLiteUpgradeOptions): Promise<void> {
                await this.sqlitePlugin.addUpgradeStatement(options);
                return;
            }
            async saveToStore(database:string) : Promise<void> {
                return await this.sqliteConnection.saveToStore(database);
            }
        }
    ```

 - Open the `services/storage.service.ts` and replace the code with

    ```ts
        import { SQLiteDBConnection } from '@capacitor-community/sqlite';
        import { Injectable} from '@angular/core';
        import { SQLiteService } from './sqlite.service';
        import { DbnameVersionService } from './dbname-version.service';
        import { UserUpgradeStatements } from '../upgrades/user.upgrade.statements';
        import { User } from '../models/user';
        import { BehaviorSubject, Observable } from 'rxjs';

        @Injectable()
        export class StorageService {
            public userList: BehaviorSubject<User[]> =
                                        new BehaviorSubject<User[]>([]);
            private databaseName: string = "";
            private uUpdStmts: UserUpgradeStatements = new UserUpgradeStatements();
            private versionUpgrades;
            private loadToVersion;
            private db!: SQLiteDBConnection;
            private isUserReady: BehaviorSubject<boolean> = new BehaviorSubject(false);

            constructor(private sqliteService: SQLiteService,
                        private dbVerService: DbnameVersionService) {
                this.versionUpgrades = this.uUpdStmts.userUpgrades;
                this.loadToVersion = this.versionUpgrades[this.versionUpgrades.length-1].toVersion;
            }
            async initializeDatabase(dbName: string) {
                this.databaseName = dbName;
                // create upgrade statements
                await this.sqliteService
                .addUpgradeStatement({  database: this.databaseName,
                                        upgrade: this.versionUpgrades});
                // create and/or open the database
                this.db = await this.sqliteService.openDatabase(
                                                    this.databaseName,
                                                    false,
                                                    'no-encryption',
                                                    this.loadToVersion,
                                                    false
                );
                this.dbVerService.set(this.databaseName,this.loadToVersion);

                await this.getUsers();
            }
            // Current database state
            userState() {
                return this.isUserReady.asObservable();
            }
            fetchUsers(): Observable<User[]> {
                return this.userList.asObservable();
            }

            async loadUsers() {
                const users: User[]= (await this.db.query('SELECT * FROM users;')).values as User[];
                this.userList.next(users);
            }
            // CRUD Operations
            async getUsers() {
                await this.loadUsers();
                this.isUserReady.next(true);
            }
            async addUser(name: string) {
                const sql = `INSERT INTO users (name) VALUES (?);`;
                await this.db.run(sql,[name]);
                await this.getUsers();
            }

            async updateUserById(id: string, active: number) {
                const sql = `UPDATE users SET active=${active} WHERE id=${id}`;
                await this.db.run(sql);
                await this.getUsers();
            }
            async deleteUserById(id: string) {
                const sql = `DELETE FROM users WHERE id=${id}`;
                await this.db.run(sql);
                await this.getUsers();
            }
        }
    ```    

 To get the current database state, we imported RxJS BehaviorSubject and Observable properties and can extract the current database state by merely subscribing to them.

 To manage the database versionning, a new service `DbnameVersionService` is required. 

 - Create it with 

    ```bash
    ng g class services/dbname-version.service
    ```

 - Open the `services/dbname-version.service.ts` file and replace the code with

    ```ts
        import { Injectable } from '@angular/core';

        @Injectable()
        export class DbnameVersionService {
            private _dbNameVersionDict: Map<string, number> = new Map();

            constructor() { }
            set(dbName: string, version: number) {
                this._dbNameVersionDict.set(dbName, version);

            }
            getVersion(dbName: string) {
                if (this._dbNameVersionDict.has(dbName)) {
                return this._dbNameVersionDict.get(dbName);
                } else {
                return -1;
                }
            }
        }
    ```

### Angular Plugin Implementation

 For Web applications, the plugin uses the Stencil component `jeep-sqlite` which implies some angular files modification

 - Open the `main.ts` file and replace the code with the following

    ```ts
        import { enableProdMode, importProvidersFrom, APP_INITIALIZER } from '@angular/core';
        import { bootstrapApplication } from '@angular/platform-browser';
        import { RouteReuseStrategy, provideRouter } from '@angular/router';
        import { IonicModule, IonicRouteStrategy } from '@ionic/angular';

        import { routes } from './app/app.routes';
        import { AppComponent } from './app/app.component';
        import { environment } from './environments/environment';
        import { defineCustomElements as pwaElements} from '@ionic/pwa-elements/loader';
        import { defineCustomElements as jeepSqlite} from 'jeep-sqlite/loader';
        import { Capacitor } from '@capacitor/core';
        import { InitializeAppService } from './app/services/initialize.app.service';
        import { SQLiteService } from './app/services/sqlite.service';
        import { StorageService } from './app/services/storage.service';
        import { DbnameVersionService } from './app/services/dbname-version.service';

        if (environment.production) {
            enableProdMode();
        }
        // --> Below only required if you want to use a web platform
        const platform = Capacitor.getPlatform();
        if(platform === "web") {
            // Web platform
            // required for toast component in Browser
            pwaElements(window);

            // required for jeep-sqlite Stencil component
            // to use a SQLite database in Browser
            jeepSqlite(window);

            window.addEventListener('DOMContentLoaded', async () => {
                const jeepEl = document.createElement("jeep-sqlite");
                document.body.appendChild(jeepEl);
                jeepEl.autoSave = true;
            });
        }
        // Above only required if you want to use a web platform <--
    
        // Define the APP_INITIALIZER factory
        export function initializeFactory(init: InitializeAppService) {
            return () => init.initializeApp();
        }

        bootstrapApplication(AppComponent, {
            providers: [SQLiteService,
                InitializeAppService,
                StorageService,
                DbnameVersionService,
                { provide: RouteReuseStrategy, useClass: IonicRouteStrategy },
                importProvidersFrom(IonicModule.forRoot({})),
                provideRouter(routes),
                {
                provide: APP_INITIALIZER,
                useFactory: initializeFactory,
                deps: [InitializeAppService],
                multi: true
                }
            ],
        });

    ``` 

 - To initialize the application, an `InitializeAppService` is used and has to be created

    ```bash
    ng g class services/initialize.app.service
    ```

 - Open the `services/initialize.app.service.ts` file and replace the code with

    ```ts
    import { Injectable } from '@angular/core';

    import { SQLiteService } from './sqlite.service';
    import { StorageService } from './storage.service';
    import { Toast } from '@capacitor/toast';

    @Injectable()
    export class InitializeAppService {
        isAppInit: boolean = false;
        platform!: string;

        constructor(
            private sqliteService: SQLiteService,
            private storageService: StorageService,
            ) {

        }

        async initializeApp() {
            await this.sqliteService.initializePlugin().then(async (ret) => {
                this.platform = this.sqliteService.platform;
                try {
                    if( this.sqliteService.platform === 'web') {
                        await this.sqliteService.initWebStore();
                    }
                    // Initialize the myuserdb database
                    const DB_USERS = 'myuserdb'
                    await this.storageService.initializeDatabase(DB_USERS);
                    // Here Initialize MOCK_DATA if required

                    // Initialize whatever database and/or MOCK_DATA you like

                    if( this.sqliteService.platform === 'web') {
                        await this.sqliteService.saveToStore(DB_USERS);
                    }

                    this.isAppInit = true;

                } catch (error) {
                    console.log(`initializeAppError: ${error}`);
                    await Toast.show({
                    text: `initializeAppError: ${error}`,
                    duration: 'long'
                    });
                }
            });
        }
    }
    ```

 - Open the `app.component.ts` file  and replace the code with

    ```ts
    import { CUSTOM_ELEMENTS_SCHEMA } from '@angular/core';
    import { Component, EnvironmentInjector, inject } from '@angular/core';
    import { IonicModule } from '@ionic/angular';
    import { CommonModule } from '@angular/common';

    @Component({
    selector: 'app-root',
    templateUrl: 'app.component.html',
    styleUrls: ['app.component.scss'],
    standalone: true,
    imports: [IonicModule, CommonModule],
    schemas: [CUSTOM_ELEMENTS_SCHEMA]
    })
    export class AppComponent {
    public environmentInjector = inject(EnvironmentInjector);

    constructor() {}
    }
    ``` 

### Users Component Implementation

 In that tutorial we use a simple UI interface for CRUD operations through the use of an Angular users component.
 It can be easily modify to implement Reactive Forms.

 - Run the below command

    ```bash
    ng g component components/users
    ``` 
 
 - Open the `components/users/users.component.ts` file and replace the code with

    ```ts
    import { Component, OnInit } from '@angular/core';
    import { CommonModule } from '@angular/common';
    import { IonicModule } from '@ionic/angular';
    import { FormsModule} from '@angular/forms';
    import { StorageService } from '../../services/storage.service';
    import { User } from '../../models/user';
    import { of, switchMap } from 'rxjs';

    @Component({
    selector: 'app-users',
    templateUrl: './users.component.html',
    styleUrls: ['./users.component.scss'],
    imports: [CommonModule, IonicModule, FormsModule],
    standalone: true,
    })
    export class UsersComponent  implements OnInit {
    newUserName = ''
    userList: User[] = []
    isWeb: any

    constructor(private storage: StorageService) {}

    ngOnInit() {
        try {
            this.storage.userState().pipe(
                switchMap(res => {
                if (res) {
                    return this.storage.fetchUsers();
                } else {
                    return of([]); // Return an empty array when res is false
                }
                })
            ).subscribe(data => {
                this.userList = data; // Update the user list when the data changes
            });

            } catch(err) {
            throw new Error(`Error: ${err}`);
            }
        }
        async createUser() {
            await this.storage.addUser(this.newUserName)
            this.newUserName = ''
            console.log(this.userList, '#users')
        }

        updateUser(user: User) {
            const active = user.active === 0 ? 1 : 0
            this.storage.updateUserById(user.id.toString(), active)
        }

        deleteUser(user: User) {
            this.storage.deleteUserById(user.id.toString())
        }
    }
    ```
 
 - Open the `components/users/users.components.html` file and replace the code with

    ```html
    <ion-content>

    <ion-row>
        <ion-col size="9">
        <ion-item>
            <ion-input
            label="Name" labelPlacement="stacked"
            placeholder="Rose Miller" [(ngModel)]="newUserName"
            >
            </ion-input>
        </ion-item>
        </ion-col>

        <ion-col size="3">
        <ion-button
        (click)="createUser()" expand="block" fill="outline">
            <ion-icon name="add" slot="icon-only"></ion-icon>
        </ion-button>
        </ion-col>
    </ion-row>

    <ion-item *ngFor="let user of userList">
        <ion-checkbox
        aria-label="Label"
        slot="start"
        ngModel="user.active"
        (ionChange)="updateUser(user)" >
        </ion-checkbox>

        <ion-label>
        {{ user.id }} - {{ user.name }} - {{ user.active }}
        </ion-label>

        <ion-button slot="end" (click)="deleteUser(user)" fill="clear" color="danger">
        <ion-icon name="trash" slot="icon-only"></ion-icon>
        </ion-button>
    </ion-item>

    </ion-content>
    ``` 

 - Now the `app-users` component can be used in the Home page. Open the `home/home.page.html` file  and replace the code with

    ```html
    <ion-header [translucent]="true">
    <ion-toolbar>
        <ion-title>
        Managing Users
        </ion-title>
    </ion-toolbar>
    </ion-header>

    <ion-content [fullscreen]="true">
    <ion-header collapse="condense">
        <ion-toolbar>
        <ion-title size="large">Managing Users</ion-title>
        </ion-toolbar>
    </ion-header>

    <app-users></app-users>
    </ion-content>
    ``` 

 - Then the `home.home.page.ts` file can be modified as below:

    ```ts
    import { Component } from '@angular/core';
    import { IonicModule } from '@ionic/angular';
    import { UsersComponent } from '../components/users/users.component';

    @Component({
    selector: 'app-home',
    templateUrl: 'home.page.html',
    styleUrls: ['home.page.scss'],
    standalone: true,
    imports: [IonicModule, UsersComponent ],
    })
    export class HomePage {
    constructor() {}
    }
    ``` 
 
### Update Config and Package.json Scripts

 - For Web application only the appID of the `capacitor.config.ts` must be amended with `YOUR_APP_ID`

 - For the `package .json` file add the two following scripts below `"build": "ng build",`,

    ```json
    "ionic:serve:before": "npm run copy:sql:wasm",
    "copy:sql:wasm": "copyfiles -u 3 node_modules/sql.js/dist/sql-wasm.wasm src/assets",
    ```

### Run the Web SQLite App

 - To run the Web app use the below command

    ```bash
    ionic serve
    ``` 

This will bring you to the `Managing Users` page

<div align="center"><br><img src="/images/ionic7-Angular-SQLite-Managing-Users.png" width="250" /></div>

On the screen-copy, ones has entered already some users.
 - To input new users
    - type a new user on the input field "Rose Miller"  under Name
    - press on the `"+"` button will add the user.

 - To update the `active`value for a user, press on the check icon on the left it will toggle the value from 1 to 0 or from 0 to 1.

 - To delete a user press on the `trash` icon.

The database location can be see in the application tab of the development tools.

<div align="center"><br><img src="/images/Ionic7-Angular-SQLite-Database-Location.png" width="300" /></div>


### Part 1 Conclusion

We have completed the Part 1 - Web application of the Ionic 7 SQLite Database CRUD App Example Tutorial using Angular and @capacitor-community/sqlite.

We learned how to implement the '@capacitor-community/sqlite' plugin in the Angular Framework using standalone components on a Web platform.

We learned how to used some basic methods of the '@capacitor-community/sqlite' plugin to create a CRUD application from scratched and store persistent SQLite data into the Web IndexedDB browser database.

Enjoy your development from there.

