## Abhängigkeiten gleich in package.json hinzufügen
-D ist die Option für die Dependency als devDependency ins 'package.json' hinzufügen
	npm i css-loader style-loader -D


## In Chrome lesbare geladene Dateien anstatt bundle.js anzeigen 
	webpack -w --devtool source-map ./entry.js bundle.js
Oder in 'webpack.config.js' 'source-map' einfügen (Im folgenden minimale Konfiguration von Webpack)
	module.exports = {
    		entry: './entry.js',
    		output : {
        	filename: 'bundle.js'
    	},
    	devtool: 'source-map'
	}

### In Chrome Breakpoint installieren
In gewünschter Codezeile (js) 
	debugger;
einfügen 
