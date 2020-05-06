public class ParserRegistrySingleton {

  
    public static ParserRegistrySingleton getInstance(){
        return ParserRegistrySingleton.Inner.instance;
    }

    private static class Inner {
        private static final ParserRegistrySingleton instance = new ParserRegistrySingleton();
    }



}
