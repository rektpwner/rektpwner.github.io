---
layout: post
title:  "Intégration de la SFML dans Qt"
date:   2024-06-08 04:57:14 -0700
categories: blog
---
Bonjour, je vous propose un petit cours sur l'intégration de la SFML dans Qt, cela fait suite aux difficultés que j'ai pu rencontrer pour effectuer cette compatibilité.

Ce cours nécessite les compétences suivantes:
- les bases en langage C++ 
- Connaissances basiques des bibliothèques Qt et SFML indépendamment

## Pourquoi vouloir intégrer la SFML dans Qt ?

Mon objectif d'origine était la création d'un jeu vidéo 2D en utilisant la SFML (Simple and Fast Multimedia Library), cependant la SFML n'a pas été conçu pour gérer les fenêtres de manière avancée. C'est pour cette raison que je souhaitais l'intégrer à Qt, qui permet de créer des fenêtres complète et plus facilement (en utilisant Qt Designer, par exemple).

Un système de fenêtrage peut être utile pour concevoir le client d'un jeu vidéo mais est indispensable quand il s'agit de concevoir le logiciel qui permet l'édition de carte du jeu vidéo.

Je vous conseil cependant d'être attentif aux licenses de ces bibliothèques dans le cas où vous souhaiteriez créer un jeu vidéo commercial.

## Outils et versions

Afin d'obtenir une intégration fonctionnelle vous pourriez utiliser les mêmes versions logiciels que ceux que j'ai utilisé dans ce cours. J'ai pour ma part installé les dernières versions disponibles de ces outils, vous pourriez tout d'abord tenter d'utiliser les versions les plus à jour de chaque logiciel puis, en cas de problème d'intégration, vous baser sur les versions de la liste ci-dessous:

- Date du test: le 8 juin 2024
- Système d'exploitation: Windows 10
- SFML: SFML-2.6.1 (GCC 13.1.0 MinGW (SEH) - 64-bit)
- Qt: Qt Creator 13.0.1 - Based on Qt 6.6.3 (MSVC 2019, x86_64)
    - Kits
        - Compiler C: MinGW 11.2.0 64-bit for C
            - Compiler path: C:\Qt\Tools\mingw1120_64\bin\gcc.exe
        - Compiler C++: MinGW 11.2.0 64-bit for C++
            - Compiler path: C:\Qt\Tools\mingw1120_64\bin\g++.exe
        - Qt version: Qt 6.7.1 MinGW 64-bit
        - CMake Tool: CMake 3.27.7 (Qt)
    - Qt versions
        - Name: Qt 6.7.1 MinGW 64-bit
        - qmake path: C:\Qt\6.7.1\mingw_64\bin\qmake.exe

## Installation

### Qt

Je vous recommande d'installer la version de Qt Online (malgré la nécessité d'avoir un compte Qt) afin que l'installation se fasse plus facilement. (en effet, la version offline nécessite de configurer manuellement un Kit dans les `Edit` > `Preferences` > `Kits`).

Télécharger Qt: [https://www.qt.io/download-qt-installer-oss](https://www.qt.io/download-qt-installer-oss)

Vous pourriez créer, compiler et lancer un programme minimal avec Qt Creator afin de confirmer que l'installation s'est bien déroulée.

Par exemple:

Fichier `main.cpp`:

{% highlight cpp %}

#include <QApplication>
#include <QPushButton>

int main(int argc, char *argv[]) {
    QApplication app(argc, argv);

    QPushButton button("Qt Hello World");
    button.resize(200, 100);
    button.show();

    return app.exec();
}

{% endhighlight %}

Fichier `.pro`:

{% highlight cpp %}

TEMPLATE = app
TARGET = QtApp
INCLUDEPATH += .

QT += core gui widgets

SOURCES += main.cpp

{% endhighlight %}

Puis le bouton `Run`.

### SFML

Vous pouvez dès à présent télécharger la SFML: [https://www.sfml-dev.org/download-fr.php](https://www.sfml-dev.org/download-fr.php)

Décompressez-le dans le dossier de votre choix ou dans le dossier de votre projet (si vous ne souhaitez faire qu'un seul projet qui utilise la SFML).

De la même manière que pour Qt, vous pouvez tester le fonctionnement de la SFML avec un code minimal

Par exemple:

Fichier `main.cpp`:

{% highlight cpp %}
#include <SFML/Graphics.hpp>

int main() {
    sf::RenderWindow window(sf::VideoMode(300, 200), "SFML Hello World");
    sf::CircleShape shape(50);
    shape.setFillColor(sf::Color::Green);

    while (window.isOpen()) {
        sf::Event event;
        while (window.pollEvent(event)) {
            if (event.type == sf::Event::Closed)
                window.close();
        }

        window.clear();
        window.draw(shape);
        window.display();
    }

    return 0;
}
{% endhighlight %}

{% highlight conf %}
g++ main.cpp -o SFMLApp -IC:\SFML\include -LC:\SFML\lib -lsfml-graphics -lsfml-window -lsfml-system
{% endhighlight %}

## Intégration de la SFML dans Qt Creator

Passons à la pratique !

Ouvrez Qt Creator

`File` > `New Project...` > `Other Project` > `Empty qmake Project` > `Choose...`.

Une fenêtre `Empty qmake Project` s'ouvre afin de configurer votre nouveau projet:
- A l'étape `Kits`, choisissez `Desktop Qt 6.7.1 MinGW 64-bit (default)`

### Code

Ajoutez ce code minimal permettant une première intégration de la SFML dans Qt, nous détaillerons le code ensuite.

Tout d'abord, la création de l'object dans le fichier `sfmlwidget.h`:

{% highlight cpp %}

#ifndef SFMLWIDGET_H
#define SFMLWIDGET_H

#include <QWidget>
#include <SFML/Graphics.hpp>

class SFMLWidget : public QWidget {
    Q_OBJECT

public:
    explicit SFMLWidget(QWidget* parent = nullptr);
    ~SFMLWidget() override = default;

protected:
    virtual void showEvent(QShowEvent* event) override;
    virtual void paintEvent(QPaintEvent* event) override;
    virtual QPaintEngine* paintEngine() const override;
    virtual void resizeEvent(QResizeEvent* event) override;

private:
    sf::RenderWindow mRenderWindow;

    void initializeSFML();
    void render();
};

#endif // SFMLWIDGET_H

{% endhighlight %}

Puis le fichier `sfmlwidget.cpp` correspondant:

{% highlight cpp %}

#include "sfmlwidget.h"

SFMLWidget::SFMLWidget(QWidget* parent) :
    QWidget(parent)
{
    setAttribute(Qt::WA_PaintOnScreen);
    setAttribute(Qt::WA_OpaquePaintEvent);
    setAttribute(Qt::WA_NoSystemBackground);

    setFocusPolicy(Qt::StrongFocus);
    setFixedSize(300, 200);
}

void SFMLWidget::showEvent(QShowEvent* event) {
    if (!mRenderWindow.isOpen()) {
        initializeSFML();
    }
    QWidget::showEvent(event);
}

void SFMLWidget::paintEvent(QPaintEvent* event) {
    (void)event;
    render();
}

QPaintEngine* SFMLWidget::paintEngine() const {
    return nullptr;
}

void SFMLWidget::resizeEvent(QResizeEvent* event) {
    if (mRenderWindow.isOpen()) {
        mRenderWindow.setSize(sf::Vector2u(width(), height()));
    }
    QWidget::resizeEvent(event);
}

void SFMLWidget::initializeSFML() {
    mRenderWindow.create(reinterpret_cast<sf::WindowHandle>(winId()));
}

void SFMLWidget::render() {
    mRenderWindow.clear(sf::Color::Black);

    sf::CircleShape shape(50.f);
    shape.setFillColor(sf::Color(100, 250, 250));

    mRenderWindow.draw(shape);

    mRenderWindow.display();
}

{% endhighlight %}

Enfin, le `main.cpp` pour utiliser notre nouvel object:

{% highlight cpp %}

#include <QApplication>
#include "sfmlwidget.h"

int main(int argc, char *argv[])
{
    QApplication app(argc, argv);

    SFMLWidget sfmlWidget;
    sfmlWidget.show();

    return app.exec();
}
{% endhighlight %}

Et le fichier `.pro` :

{% highlight conf %}
QT += core gui widgets

CONFIG += c++11

INCLUDEPATH += "C:\<your path>\SFML-2.6.1\include"

LIBS += -L"C:\<your path>\SFML-2.6.1\lib" -lsfml-graphics-s -lsfml-window -lsfml-system

SOURCES += main.cpp \
           sfmlwidget.cpp

HEADERS += sfmlwidget.h

{% endhighlight %}

### Explication

Nous allons détailler ce code.

Tout d'abord, intéressons nous au fichier `sfmlwidget.h`:

La ligne suivante: `class SFMLWidget : public QWidget {` permet de créer un object `SFMLWidget` qui hérite de l'object `QWidget` (la classe de base de tous les widgets Qt), cela permet de récupérer l'ensemble des méthodes Qt utiles afin de pouvoir placer correctement cet élément dans la fenêtre principale: [https://doc.qt.io/qt-6/qwidget.html](https://doc.qt.io/qt-6/qwidget.html)

Ainsi la SFML peut gérer le rendu graphique tout en laissant Qt gérer l'interface utilisateur.

La macro `Q_OBJECT` est ignoré dans le cas où vous utilisez `qmake`, je l'ai ajouté pour éviter d'avoir des problèmes avec ce code dans le cas où vous ne souhaitez pas utiliser `qmake` pour compiler ce code, je vous conseil de `rebuild` entièrement le project dans le cas où vous ajouteriez cette macro après une première compilation et que vous n'utilisez pas `qmake`.
Cette macro permet d'ajouter des fonctionnalités de type `signals et slots` ou autres propriétés avancées de Qt, plus d'information ici: [https://doc.qt.io/qt-6/moc.html](https://doc.qt.io/qt-6/moc.html)

La suite:
{% highlight cpp %}

protected:
    virtual void showEvent(QShowEvent* event) override;
    virtual void paintEvent(QPaintEvent* event) override;
    virtual QPaintEngine* paintEngine() const override;
    virtual void resizeEvent(QResizeEvent* event) override;

{% endhighlight %}

Le mot-clef `virtual` aurait pu être omis sur chaque méthode, cependant, pour obtenir un code plus logique, je les ai volontairement laissé.

Cette liste de méthodes sont fournis grâce à l'héritage de QWidget, toute l'astuce de l'intégration de la SFML dans Qt se trouve ici.
Nous souhaitons effectuer une redéfinition (et non une surcharge) de ces méthodes présente dans l'object parent hérité: `QWidget`
Le mot-clef `override` permet de nous prévenir le compilateur que nous souhaitons exclusivement faire une redéfinition, cela permet de remonter une erreur lors de la compilation dans le cas où les deux méthodes ont le même nom (enfant et parent) mais que la signature de la méthode change (donc une surcharge non-voulue). Il est préférable d'obtenir une erreur à la compilation plutôt que d'avoir un comportement inattendu et difficile à debug lors du lancement.

L'objectif est de laisser la main à la SFML pour gérer ces méthodes:

`virtual void showEvent(QShowEvent* event) override;`
- Cela sert à initialiser la fenêtre SFML uniquement la première fois que le widget est affiché et évite une initialisation prématurée ou multiple de la fenêtre SFML.

`virtual void paintEvent(QPaintEvent* event) override;`
- Cela sert à personnaliser le rendu du widget en utilisant la SFML et permet de dessiner des éléments de la SFML uniquement lorsque le widget a besoin d'être redessiné.

`virtual QPaintEngine* paintEngine() const override;`
- Cela sert à indiquer à Qt que ce widget utilise un moteur de rendu personnalisé (ici, la SFML) et qu'il ne faut pas utiliser le moteur de rendu par défaut, ce qui empêche les artefacts graphiques et les conflits entre le rendu de Qt et celui de la SFML.

`virtual void resizeEvent(QResizeEvent* event) override;`
- Cela sert à ajuster la taille de la vue SFML par rapport au redimensionnement du widget afin de maintenir les proportions et l'échelle de la vue SFML, assurant que le contenu graphique est correctement redimensionné.

Passons maintenant au fichier `sfmlwidget.cpp`:

Et on commence avec le constructeur:

`setAttribute(Qt::WA_PaintOnScreen);`
- Cette ligne indique à Qt que le widget va gérer son propre rendu directement à l'écran car par défaut, Qt utilise un système de double buffering pour dessiner sur des widgets afin de réduire le scintillement et les artefacts visuels. En désactivant ce comportement de Qt cela permet à la SFML de dessiner directement sur l'écran. C'est une étape nécessaire pour intégrer des bibliothèques graphiques externes comme la SFML qui gère son propre rendu et sans bénéficier du double buffering de Qt.

`setAttribute(Qt::WA_OpaquePaintEvent);`
- Cela indique à la fenêtre Qt que le widget gère l'intégralité de son propre contenu et que Qt n'a pas besoin d'effacer l'arrière-plan avant de dessiner. Ce qui évite un effacement inutile de l'arrière-plan et qui pourrait causer des artefacts visuels ou des conflits avec le rendu de SFML, améliore également les performances.

`setAttribute(Qt::WA_NoSystemBackground);`
- Ceci va permettre d'empêche Qt de dessiner l'arrière-plan du widget (comportement par défaut de Qt) ce qui pourrait interférer avec le rendu de SFML, cela afin que ce soit la SFML qui assure un contrôle complet sur ce qui est dessiné dans le widget, sans interférence de Qt.

`setFocusPolicy(Qt::StrongFocus);`
- Ici se définit la politique de gestion du focus pour le widget ce qui va permettre au widget de recevoir le focus à la fois via un événement clavier que souris, particulièrement utile pour les applications interactives comme les jeux vidéos, par exemple.

La méthode `showEvent` :

`if (!mRenderWindow.isOpen()) {`
- garantit que l'initialisation de la fenêtre SFML ne se fait qu'une seule fois, même si la méthode `showEvent` est appelée plusieurs fois, permet d'éviter des réinitialisations inutiles ou des conflits d'initialisation.

La ligne `mRenderWindow.create(reinterpret_cast<sf::WindowHandle>(winId()));` est très particulière:
- Pour commencer `winId()` qui est une méthode de `QWidget` retourne l'identifiant natif de la fenêtre Qt (aussi appelé un handle sous Windows) qui est un identifiant unique d'une fenêtre, il est géré par le système d'exploitation (chaque système d'exploitation (Windows, macOS, Linux...) a son propre mécanisme pour gérer les fenêtres et autres ressources graphiques).
- Ensuite, `mRenderWindow.create` qui attend une valeur de type `sf::WindowHandle` (d'où le cast) qui sert à préparer le widget SFML pour recevoir des commandes de rendu et pour être affichée à l'écran.

Nous pouvons maintenant passer au lancement de l'application.

### Lancement

Lorsque vous effectuez une modification sur le fichier `.pro`, je vous conseil de faire un clic droit sur votre projet dans Qt Creator puis `Clean`, puis clic droit à nouveau et `Run qmake`.

Lancez le programme et vous obtiendrez plusieurs erreurs du type:

![(Image not found) System error - The code execution cannot proceed because Qt6Widgets.dll was not found. Reinstalling the program may fix this problem]({{ site.baseurl }}/assets/ErrorQt6Widgets.dll.png)

![(Image not found) System error - The code execution cannot proceed because Qt6Core.dll was not found. Reinstalling the program may fix this problem]({{ site.baseurl }}/assets/ErrorQt6Core.dll.png)

[...]

Ces erreurs lors de l'ouverture de l'application signifie que l'application utilise des fonctions externes et qu'il a été compilé *dynamiquement*, donc que la résolution de ces fonctions (la recherche du code de ces fonctions dans les .dll) se fait pendant l'exécution du programme.

Il y a deux solutions pour résoudre ce problème:
- La première solution, continuer à compiler dynamiquement l'application et placer les fichiers .dll manquants dans le même dossier que celui de l'application.
- La seconde solution, compiler statiquement l'application

Je ne détaillerai pas ici tous les avantages et inconvénients de ces deux méthodes, n'hésitez pas à faire vos propres recherches pour en apprendre davantage.

#### Première solution - la compilation dynamique

Vous pourriez chercher les .dll manquantes une à une et les placer dans le dossier courant de votre application, mais cela serait long et ennuyeux à réaliser. Heureusement Qt nous fournit un outil permettant de copier automatiquement toutes les .dll de Qt nécessaires dans le dossier courant de notre application.

Cette application se nomme `windeployqt`, dont voici la documentation officielle: [https://doc.qt.io/qt-6/windows-deployment.html](https://doc.qt.io/qt-6/windows-deployment.html)

L'utilisation dans un terminal est assez simple:

{% highlight plaintext %}

windeployqt <chemin vers l'application .exe>

{% endhighlight %}

Vous pourriez également opter pour une alternative: [https://wiki.qt.io/CQtDeployer](https://wiki.qt.io/CQtDeployer)

#### Deuxième solution - la compilation statique

Cette méthode permet d'inclure directement le code des fonctions externes à l'intérieur de l'application, ainsi il n'est plus nécessaire d'avoir des fichiers .dll dans le même dossier afin de faire fonctionner l'application, le fichier .exe seul suffit.

Pour réaliser cela, nous allons modifier le contenu du fichier `.pro`:

{% highlight conf %}
QT += core gui widgets

CONFIG += c++11

DEFINES += SFML_STATIC # Nouvelle ligne

INCLUDEPATH += "C:\<your path>\SFML-2.6.1\include"

# Nouvelles lignes
LIBS += -L"C:\<your path>\SFML-2.6.1\lib" \
    -lsfml-graphics-s \
    -lsfml-window-s \
    -lsfml-system-s \
    -lopengl32 \
    -lwinmm \
    -lgdi32 \
    -lfreetype \
    -lopenal32 \
    -lvorbisfile \
    -lvorbis \
    -logg \
    -lws2_32

QMAKE_LFLAGS += -static-libgcc -static-libstdc++ # Nouvelle ligne

SOURCES += main.cpp \
           sfmlwidget.cpp

HEADERS += sfmlwidget.h

{% endhighlight %}

La liste des dépendances est longue mais nécessaire.

N'oubliez pas `Clean` et `Run qmake`.

Vous devriez maintenant être capable de lancer votre application et d'admirer le résultat.

![(Image not found) Fenêtre qui dessine un rond en SFML à l'intérieur d'une fenêtre Qt]({{ site.baseurl }}/assets/SFMLWithQt1.png)

## Intégration de la SFML dans Qt Designer

Ouvrez Qt Creator

`File` > `New Project...` > `Application (Qt)` > `Qt Widget Application` > `Choose...`.

Une fenêtre `Qt Widget Application` s'ouvre afin de configurer votre nouveau projet:
- A l'étape `Build System`, choisissez `qmake`
- A l'étape `Kits`, choisissez `Desktop Qt 6.7.1 MinGW 64-bit (default)`

### Code

Fichier `mainwindow.h`:

{% highlight cpp %}

#ifndef MAINWINDOW_H
#define MAINWINDOW_H

#include <QMainWindow>

QT_BEGIN_NAMESPACE
namespace Ui {
class MainWindow;
}
QT_END_NAMESPACE

class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    MainWindow(QWidget *parent = nullptr);
    ~MainWindow();

private:
    Ui::MainWindow *ui;
};
#endif // MAINWINDOW_H

{% endhighlight %}

Fichier `mainwindow.cpp`:

{% highlight cpp %}

#include "mainwindow.h"
#include "ui_mainwindow.h"

MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);
}

MainWindow::~MainWindow()
{
    delete ui;
}

{% endhighlight %}

Fichier `sfmlwidget.h`:

{% highlight cpp %}

#ifndef SFMLWIDGET_H
#define SFMLWIDGET_H

#include <QWidget>
#include <SFML/Graphics.hpp>

class SFMLWidget : public QWidget {
    Q_OBJECT

public:
    explicit SFMLWidget(QWidget* parent = nullptr);
    ~SFMLWidget() override = default;

protected:
    virtual void showEvent(QShowEvent* event) override;
    virtual void paintEvent(QPaintEvent* event) override;
    virtual QPaintEngine* paintEngine() const override;
    virtual void resizeEvent(QResizeEvent* event) override;

private:
    sf::RenderWindow mRenderWindow;

    void initializeSFML();
    void render();
};

#endif // SFMLWIDGET_H

{% endhighlight %}

Fichier `sfmlwidget.cpp`:

{% highlight cpp %}

#include "sfmlwidget.h"

SFMLWidget::SFMLWidget(QWidget* parent) :
    QWidget(parent)
{
    setAttribute(Qt::WA_PaintOnScreen);
    setAttribute(Qt::WA_OpaquePaintEvent);
    setAttribute(Qt::WA_NoSystemBackground);

    setFocusPolicy(Qt::StrongFocus);
}

void SFMLWidget::showEvent(QShowEvent* event) {
    if (!mRenderWindow.isOpen()) {
        initializeSFML();
    }
    QWidget::showEvent(event);
}

void SFMLWidget::paintEvent(QPaintEvent* event) {
    (void)event;
    render();
}

QPaintEngine* SFMLWidget::paintEngine() const {
    return nullptr;
}

void SFMLWidget::resizeEvent(QResizeEvent* event) {
    if (mRenderWindow.isOpen()) {
        mRenderWindow.setSize(sf::Vector2u(width(), height()));
    }
    QWidget::resizeEvent(event);
}

void SFMLWidget::initializeSFML() {
    mRenderWindow.create(reinterpret_cast<sf::WindowHandle>(winId()));
}

void SFMLWidget::render() {
    mRenderWindow.clear(sf::Color::Black);

    sf::CircleShape shape(50.f);
    shape.setFillColor(sf::Color(100, 250, 250));

    mRenderWindow.draw(shape);

    mRenderWindow.display();
}

{% endhighlight %}

Vous devez maintenant utiliser l'interface de Qt Designer pour placer votre widget, rendez-vous dans le fichier `mainwindow.ui` puis cherchez le `Containers` qui se nomme `Widget`, effectuez un glissez-déposez dans votre fenêtre.

Une fois placée, faites un `clic-droit` sur celui-ci > `Promote to...` > `SFMLWidget`.

### Explication

Le code est assez similaire au précédent à l'exception de la ligne: `setFixedSize(300, 200);` qui a été enlevée afin que le rendu de la SFML s'adapte à la taille du widget que vous avez défini dans le fichier `mainwindow.ui`. Ce widget est donc toujours de taille fixe mais est maintenant configurable directement dans Qt Designer. Nous verrons plus tard comme créer un widget `SFMLWidget` responsive (c'est-à-dire qui dépend et s'adapte à la taille de la fenêtre Qt).

### Lancement

Vous pouvez maintenant lancer le programme est observer le résultat obtenu (j'ai ajouté des éléments au menuBar et un bouton en plus du widget, pour l'exemple).

![(Image not found) Fenêtre qui dessine un rond en SFML à l'intérieur d'une fenêtre Qt créée via Qt Designer]({{ site.baseurl }}/assets/SFMLWithQt2.png)

## Gestion des vues SFML dans Qt

Félicitation ! Vous avez crée une fenêtre sur Qt Designer qui intègre un widget `SFMLWidget`, mais maintenant vous souhaitez aller plus loin ?

Je vous propose de permettre au widget `SFMLWidget` de s'adapter à la taille de la fenêtre sans déformer le rendu de la SFML.

Vous pourrez trouver plus d'information sur le site officiel: [https://doc.qt.io/qt-6/layout.html](https://doc.qt.io/qt-6/layout.html)

Nous allons donc commencer par rendre notre fenêtre responsive, pour cela selectionnez la fenêtre principale via l'interface de Qt Designer (fichier: `mainwindow.ui`), en cliquant sur la fenêtre ou en selectionnant l'objet `MainWindow` sur le fenêtre lattérale `Object Inspector`.

![(Image not found) Object Inspector de Qt Designer]({{ site.baseurl }}/assets/SFMLWithQt3.png)

Puis utilisez les icônes situés sur le haut de votre éditeur Qt Designer pour appliquer un Lay Out Horizontally ou Lay Out Vertically (celui de votre choix).

![(Image not found) Layout de Qt Designer]({{ site.baseurl }}/assets/SFMLWithQt4.png)

Les éléments présents dans votre fenêtre devrait s'aligner automatiquement par rapport aux proportions de la fenêtre.

![(Image not found) Application de l'alignement de Qt Designer]({{ site.baseurl }}/assets/SFMLWithQt5.png)

Vous pouvez maintenant lancer le programme et analyser le comportement de la fenêtre lorsque vous tentez de la redimensionner.

![(Image not found) Déformation du rendu de la SFML]({{ site.baseurl }}/assets/SFMLWithQt6.png)

Le rendu de la SFML se déforme !

Lors de mes tentatives d'intégrations, j'ai alors tenté d'ajouter dans le fichier `sfmlwidget.h` un nouvel attribut private: `sf::View mView;` mais la seule présence de cette ligne dans l'objet (même sans l'utiliser) suffit à faire crash l'application Qt lors de son lancement.

Voici une solution permettant de résoudre ce problème de compatibilité.

Retournez dans Qt Designer, dans la fenêtre lattérale `Object Inspector`, selectionnez le widget `SFMLWidget`, cherchez sa prorpriété `objectName` dans la fenêtre lattérale `Property Editor` et renommez-le `MyWidget`.

Nous pouvons maintenant modifier le code de 2 fichiers du projet: `sfmlwidget.h` et `sfmlwidget.cpp`.

### Code

Fichier `sfmlwidget.h`:

{% highlight cpp %}

#ifndef SFMLWIDGET_H
#define SFMLWIDGET_H

#include <QWidget>
#include <SFML/Graphics.hpp>

class SFMLWidget : public QWidget {
    Q_OBJECT

public:
    explicit SFMLWidget(QWidget* parent = nullptr);
    ~SFMLWidget() override;

protected:
    void showEvent(QShowEvent* event) override;
    void paintEvent(QPaintEvent* event) override;
    QPaintEngine* paintEngine() const override;
    void resizeEvent(QResizeEvent* event) override;

private:
    sf::RenderWindow* mRenderWindow;
    //sf::View mView;
    sf::View getAdjustedView();

    void initializeSFML();
    void render();
};

#endif // SFMLWIDGET_H

{% endhighlight %}

Fichier `sfmlwidget.cpp`:

{% highlight cpp %}

#include "sfmlwidget.h"

SFMLWidget::SFMLWidget(QWidget* parent) :
    QWidget(parent),
    mRenderWindow(nullptr)
{
    setAttribute(Qt::WA_PaintOnScreen);
    setAttribute(Qt::WA_OpaquePaintEvent);
    setAttribute(Qt::WA_NoSystemBackground);

    setFocusPolicy(Qt::StrongFocus);
    //setFixedSize(300, 200);
}

SFMLWidget::~SFMLWidget() {
    if (mRenderWindow) {
        delete mRenderWindow;
    }
}

void SFMLWidget::showEvent(QShowEvent* event) {
    if (!mRenderWindow) {
        initializeSFML();
    }
    QWidget::showEvent(event);
}

void SFMLWidget::paintEvent(QPaintEvent* event) {
    (void)event;
    render();
}

QPaintEngine* SFMLWidget::paintEngine() const {
    return nullptr;
}

void SFMLWidget::resizeEvent(QResizeEvent* event) {
    if (mRenderWindow) {
        mRenderWindow->setSize(sf::Vector2u(width(), height()));
        mRenderWindow->setView(getAdjustedView());
    }
    QWidget::resizeEvent(event);
}

void SFMLWidget::initializeSFML() {
    mRenderWindow = new sf::RenderWindow(reinterpret_cast<sf::WindowHandle>(winId()));
    mRenderWindow->setView(getAdjustedView());
}

sf::View SFMLWidget::getAdjustedView() {
    QWidget* myWidget = parentWidget()->findChild<QWidget*>("MyWidget");
    if (!myWidget) {
        // Fallback if widget is not found
        return sf::View(sf::FloatRect(0, 0, 300, 200));
    }

    // Get the size of the widget
    float widgetWidth = myWidget->width();
    float widgetHeight = myWidget->height();

    sf::View view;
    float windowRatio = static_cast<float>(width()) / static_cast<float>(height());
    float viewRatio = widgetWidth / widgetHeight;

    if (windowRatio > viewRatio) {
        // Window is wider than the view
        float newWidth = widgetHeight * windowRatio;
        view.setSize(newWidth, widgetHeight);
    } else {
        // Window is taller than the view
        float newHeight = widgetWidth / windowRatio;
        view.setSize(widgetWidth, newHeight);
    }
    view.setCenter(widgetWidth / 2.f, widgetHeight / 2.f); // Center the view
    return view;
}

void SFMLWidget::render() {
    mRenderWindow->clear(sf::Color::Black);

    sf::CircleShape shape(50.f);
    shape.setFillColor(sf::Color(100, 250, 250));
    mRenderWindow->draw(shape);

    mRenderWindow->display();
}

{% endhighlight %}

### Explication

La vue est actualisée à chaque fois que la fenêtre change de taille, une récupération de la nouvelle taille du widget est effectuée pour une mise à l'échelle afin de maintenir les proportions, cela évite les déformations. Enfin, il faut centrer la vue pour ne pas avoir le rendu de la SFML décalé ou coupé.

### Lancement

Lancez l'application, redimensionnez-la, elle s'adapte parfaitement à ses nouvelles dimensions.

C'est la fin de ce cours, j'espère qu'il vous a été utile.