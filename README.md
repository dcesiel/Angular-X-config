# Reading data before application startup in Angular 2

In this demonstration I will show you how to read data in Angular2 final release before application startup. You can use it to read configuration files like you do in other languages like Java, Python, Ruby, Php.

This is how the demonstration will load data:

a) It will read an env file named 'env.json'. This file indicates what is the current working environment. Options are: 'production' and 'development';

b) It will read a config JSON file based on what is found in env file. If env is "production", the file is 'config.production.json'. If env is "development", the file is 'config.development.json'.

All these reads will be done before Angular2 starts up the application.

It assumes you already have a working application, with a module and everything set up.

### In Your Module

Open your existing module and add the following two lines to your list of providers.

```typescript
import { APP_INITIALIZER } from '@angular/core';
import { AppConfig }       from './app.config';
import { HttpModule }      from '@angular/http';

...

@NgModule({
    imports: [
        ...
        HttpModule
    ],
    ...
    providers: [
        ...
        AppConfig,
        { provide: APP_INITIALIZER, useFactory: (config: AppConfig) => () => config.load(), deps: [AppConfig], multi: true }
    ],
    ...
});
```

The first line makes AppConfig class available to Angular2.

The second line uses APP_INITIALIZER to execute Config.load() method before application startup. The 'multi: true' is being used because an application can have more than one line of APP_INITIALIZER.

Make sure you set "HttpModule" in "imports" section if you want to make http calls using Angular2 built in Http library.

### In app.config.ts

Create a class AppConfig and name the file 'app.config.ts' (you can use a name of your choice).

This is the place we will do the reading of env and config files. The data of both files will be stored in the class so we can retrieve it later.

Note that native Angular Http library is used to read the json files.

```typescript
import { Inject, Injectable } from '@angular/core';
import { Http } from '@angular/http';
import { Observable } from 'rxjs/Rx';

@Injectable()
export class AppConfig {

    private config: Object = null;
    private env:    Object = null;

    constructor(private http: Http) {

    }

    /**
     * Use to get the data found in the second file (config file)
     */
    public getConfig(key: any) {
        return this.config[key];
    }

    /**
     * Use to get the data found in the first file (env file)
     */
    public getEnv(key: any) {
        return this.env[key];
    }

    /**
     * This method:
     *   a) Loads "env.json" to get the current working environment (e.g.: 'production', 'development')
     *   b) Loads "config.[env].json" to get all env's variables (e.g.: 'config.development.json')
     */
    public load() {
        return new Promise((resolve, reject) => {
            this.http.get('env.json').map( res => res.json() ).catch((error: any):any => {
                console.log('Configuration file "env.json" could not be read');
                resolve(true);
                return Observable.throw(error.json().error || 'Server error');
            }).subscribe( (envResponse) => {
                this.env = envResponse;
                let request:any = null;

                switch (envResponse.env) {
                    case 'production': {
                        request = this.http.get('config.' + envResponse.env + '.json');
                    } break;

                    case 'development': {
                        request = this.http.get('config.' + envResponse.env + '.json');
                    } break;

                    case 'default': {
                        console.error('Environment file is not set or invalid');
                        resolve(true);
                    } break;
                }

                if (request) {
                    request
                        .map( res => res.json() )
                        .catch((error: any) => {
                            console.error('Error reading ' + envResponse.env + ' configuration file');
                            resolve(error);
                            return Observable.throw(error.json().error || 'Server error');
                        })
                        .subscribe((responseData) => {
                            this.config = responseData;
                            resolve(true);
                        });
                } else {
                    console.error('Env config file "env.json" is not valid');
                    resolve(true);
                }
            });

        });
    }
}
```

See that we used resolve() in all scenarios because we don't want the application to crash if any problem is found in the configuration files. If you prefer, you can set error scenarios to reject().

### In env.json
This is the place you will configure the current development environment. Allowed values are 'development' and 'production'.

```json
{
    "env": "development"
}
```
You may add this file to .gitignore to your convenience.

### In config.development.json
This is the place you will configure development config variables. You can add as many variables you want in this JSON file.

```json
{
    "host": "localhost"
}
```
You may add this file to .gitignore to your convenience.

### In config.production.json
This is the place you will write production config variables. You can add as many variables you want in this JSON file.

```json
{
    "host": "112.164.12.21"
}
```
You may add this file to .gitignore to your convenience.

### In Any Angular2 class
Example of how we read the values previously loaded from both files. In this case, we are reading the 'host' variable from config file and 'env' from the env file.

```typescript
import { AppConfig } from './app.config';

export class AnyClass {
    constructor(private config: AppConfig) {
        // note that AppConfig is injected into a private property of AnyClass
    }
    
    myMethodToGetHost() {
        // will print 'localhost'
        let host:string = config.get('host');
    }
    
    myMethodToGetCurrentEnv() {
        // will print 'development'
        let env: string = config.getEnv('env');
    }
}
```
