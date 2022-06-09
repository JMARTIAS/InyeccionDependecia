# Inyección de dependencias Dagger Hilt
para iniciar un proyecto con dagger hilt hay que implementar unas dependencias, estas son las de Hilt e incluyen las que son de Dagger 
```gradle
 //Dagger - Hilt
    implementation "com.google.dagger:hilt-android:2.42"
    kapt "com.google.dagger:hilt-compiler:2.42"
```

tambien se debe poner en plugins (un poco mas arriba)
```gradle
//kotlin anotation processing
id 'kotlin-kapt'
id 'dagger.hilt.android.plugin'
```

también en el gradle del proyecto hay que poner este classpath para no tener problemas con 
```gradle
classpath "com.google.dagger:hilt-android-gradle-plugin:2.42"
```
despues se hace el sync now

se necesita crear una aplication class al usar hilt, ya que necesita, establecer varias clases
por detrás para nosotros, y por eso hilt necesita saber cuales son las clases de la aplicación,
cual es el contexto de la aplicació
por detrás para nosotros, y por eso hilt necesita saber cuales son las clases de la aplicación, 
lo necesita para inyectar nuestras dependencias. Hilt necesita saber donde esta nuestra aplication class 

asi que mi clase recien creada hereda de Application() y le pongo la notacion
@HiltAndroidApp, esto es todo lo que Hilt necesita para generar las clases necesarias para application

```kotlin
@HiltAndroidApp
class MyApplication : Application()
```

tambien hay que declarar esta clase en el manifest
```xml
<application
        android:name=".MyApplication"
```
 
 con esto es suficiente para inyectar dependencias con Dagger Hilt, pero necesitamos dependencias para inyectar
 para eso creamos una clase llamada AppModule, que es un object, en Dagger hilt creas 1 o varios modulos
 que tienen las dependencias, que viven mientras nuestra aplicación viva, esto podria ser una instancia de retrofit
  una instancia de room.
  
  Con Dagger hilt podemos hacer un scope de nuestras dependencias, se puede crear un modulo que solo viva lo que vive una actividad 
  
  entonces creamos nuestro objeto y le ponemos la anotacion @Module y  tenemos que decirle 
  cual es el scope que queremos en que viva que se hace con @InstallIn(SingletonComponent::class), que dice que viven 
  mientras viva la app 
  tb hay ActivityComponent, que hace que las dependencias vivan mientras viva esta activity,
  hay FragmentComponent
  ViewComponent
  
  tenemos que darle a dagger hilt un molde de lo que queremos inyectar. aqui declaramos nuestros strings
  solo declaramos funciones
  
  y la convencioón de nombre es provide y que proveemos como y que devolvemos, lo que queremos proveer, osea que queremos inyectar en nuestra clase
  y hay que poner 2 anotaciones mas @Singleton, la que hace que sea un Singleton, si no lo ponemos cada vez que se inyecte este componente se creará una nueva instancia
  
  La otra anotacion es @Provides
  ```kotlin
  @Module
@InstallIn(SingletonComponent::class)
object AppModule {

    @Singleton
    @Provides
    @Named("String1")
    fun provideTestString1() = "This is a string we will inject"

}
```

Para inyectarla en nuestra main activity tambien se necesita una anotacion @AndroidEntryPoint
y si solo quiero inyectarlo en un fragment, se pone en este y su actividad padre 

se decalara una lateinit var que sea del tipo que voy a inyectar (creo)
y le ponemos la notacion @Inject
busca en todos los modulos el tipo que corresponde y lo asigna a la variable.
Si tengo 2 del mismo tipo no sabe cual inyectar y le damos la anotacion @Named("NombreQueQuiera") y en la activity le ponemos
la anotacion @Named("NombreQueQuiera") tambien

```kotlin

@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    @Inject
    @Named("String1")
    lateinit var testString: String

    private val viewModel: TestViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        Log.d("MainActivity", "Test String from MainActivity: $testString")
        viewModel
    }
}
  
```

Singeton solo funciona en el Scope de application, no tiene sentido que sea en activity, asi que ahi existe la anotacion @ActivityScoped
eso quiere decir, tenemos esta dependencia que vive mientras viva nuestra activity, pero solo hay una instancia de este
asi que si lo inyectamos multiples veces en nuestra activity, solo habra una instancia, no creara nuevas

el contexto es algo especial en dagger hilt

hay qu poner la notacion @ApplicationContext para obtenerlo

a veces necesitamos dependencias creadas antes y las definimos como un parametro

```kotlin
@Module
@InstallIn(ActivityComponent::class)
object MainModule {

    @ActivityScoped
    @Provides
    @Named("String2")
    fun provideTestString2(
        @ApplicationContext context: Context,
        @Named("String1") testString1: String
    ) = "${context.getString(R.string.string_to_inject)} - $testString1"
}
```



## Inyectar Dependencias en ViewModel

se deben inyectar dentro del constructor así que se hacia con un factory, ahora no es necesario hacerlo así
dentro del constructor se pone @ViewModelInject, que hace el Factory detrás

```kotlin
class TestViewModel @ViewModelInject constructor(
    @Named("String2") testString2: String
) : ViewModel() {

    init {
        Log.d("ViewModel", "Test String from ViewModel: $testString2")
    }
}
```

para inyectarlo en la activity, no se usa inject solo se usa una private val 

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    @Inject
    @Named("String1")
    lateinit var testString: String

    private val viewModel: TestViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        Log.d("MainActivity", "Test String from MainActivity: $testString")
        viewModel
    }
}
```

aca un ejemplo de Module

```kotlin
@Module
@InstallIn(SingletonComponent::class)
class AlbumModule {
    @Provides
    @Singleton
    fun provideRetrofitApi(): PhotoAlbumAPI {
        return Retrofit.Builder()
            .baseUrl(BuildConfig.BASE_URL)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(PhotoAlbumAPI::class.java)
    }

    @Provides
    @Singleton
    fun provideGetAlbumUseCase(repository: AlbumRepository): GetAlbumUseCase {
        return GetAlbumUseCase(repository)
    }

    @Provides
    @Singleton
    fun provideGetAlbumRepository(dataSource: AlbumRemoteDataSource): AlbumRepository{
        return AlbumRepositoryImpl(dataSource)
    }

    @Provides
    @Singleton
    fun provideAlbumRemoteDataSource(api: PhotoAlbumAPI): AlbumRemoteDataSource{
        return AlbumRemoteDataSource(api)
    }

}
```
