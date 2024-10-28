import java.util.ArrayList;
import java.util.List;
import java.awt.*;
import java.awt.image.BufferedImage;
import java.io.*;
import java.util.*;
import javax.imageio.ImageIO;

public class Proyecto {
    private Set<Integer> states = new HashSet<>();
    private Set<Character> alphabet = new HashSet<>();
    private Map<String, Set<Integer>> afnTransitions = new HashMap<>();
    private Map<String, Integer> afdTransitions = new HashMap<>();
    private int startState;
    private Set<Integer> acceptStates = new HashSet<>();
    private List<String> validationHistory = new ArrayList<>(); // Historial de validación
    private int executionCounter = 0; // Contador de ejecuciones

    // Constructor que inicia el estado inicial y final
    public Proyecto() {
        startState = 0;
        acceptStates.add(1);
    }

    // Método para agregar transiciones
    public void addTransition(int fromState, char symbol, Set<Integer> toStates) {
        String key = fromState + "," + symbol;
        afnTransitions.putIfAbsent(key, new HashSet<>());
        afnTransitions.get(key).addAll(toStates);
    }

    // Generación básica del AFN desde una expresión regular simple
    public void generateAFNFromRegex(String regex) {
        int currentState = 0;
        for (int i = 0; i < regex.length(); i++) {
            char ch = regex.charAt(i);
            if (ch == 'a' || ch == 'b') {
                Set<Integer> nextState = new HashSet<>(Collections.singleton(currentState + 1));
                addTransition(currentState, ch, nextState);
                alphabet.add(ch);
                currentState++;
                states.add(currentState);
            }
        }
        acceptStates.add(currentState);
        System.out.println("AFN generado desde la expresión regular: " + regex);
    }

    // Conversión del AFN a AFD
    public void convertToAFD() {
        Map<Set<Integer>, Integer> newStates = new HashMap<>();
        Queue<Set<Integer>> queue = new LinkedList<>();
        Set<Integer> initialState = epsilonClosure(Collections.singleton(startState));
        
        queue.add(initialState);
        newStates.put(initialState, 0);
        
        int stateId = 1;
        while (!queue.isEmpty()) {
            Set<Integer> currentSet = queue.poll();
            int currentId = newStates.get(currentSet);
            
            for (char symbol : alphabet) {
                Set<Integer> nextSet = epsilonClosure(move(currentSet, symbol));
                
                if (!nextSet.isEmpty() && !newStates.containsKey(nextSet)) {
                    newStates.put(nextSet, stateId++);
                    queue.add(nextSet);
                    
                    if (containsAcceptState(nextSet)) {
                        acceptStates.add(newStates.get(nextSet));
                    }
                }
                
                if (!nextSet.isEmpty()) {
                    afdTransitions.put(currentId + "," + symbol, newStates.get(nextSet));
                }
            }
        }
        
        this.states = new HashSet<>(newStates.values());
        this.startState = newStates.get(initialState);
        System.out.println("Conversión a AFD completada.");
    }

    // Validación de una cadena en el AFD
    public boolean validateString(String input) {
        int currentState = startState;
        
        for (char symbol : input.toCharArray()) {
            String key = currentState + "," + symbol;
            if (afdTransitions.containsKey(key)) {
                currentState = afdTransitions.get(key);
            } else {
                return false;
            }
        }
        
        return acceptStates.contains(currentState);
    }

    // Cierre epsilon
    private Set<Integer> epsilonClosure(Set<Integer> states) {
        Stack<Integer> stack = new Stack<>();
        Set<Integer> closure = new HashSet<>(states);
        stack.addAll(states);
        
        while (!stack.isEmpty()) {
            int state = stack.pop();
            String key = state + ",$";
            
            if (afnTransitions.containsKey(key)) {
                for (int s : afnTransitions.get(key)) {
                    if (!closure.contains(s)) {
                        closure.add(s);
                        stack.push(s);
                    }
                }
            }
        }
        
        return closure;
    }

    // Función para mover el estado en función del símbolo
    private Set<Integer> move(Set<Integer> states, char symbol) {
        Set<Integer> result = new HashSet<>();
        
        for (int state : states) {
            String key = state + "," + symbol;
            if (afnTransitions.containsKey(key)) {
                result.addAll(afnTransitions.get(key));
            }
        }
        
        return result;
    }

    private boolean containsAcceptState(Set<Integer> states) {
        for (int state : states) {
            if (acceptStates.contains(state)) {
                return true;
            }
        }
        return false;
    }

    // Guardar el AFD en un archivo
    public void saveToFile(String filename) {
        try (PrintWriter writer = new PrintWriter(new FileWriter(filename, true))) { // 'true' para agregar sin borrar
            executionCounter++;
            writer.println("Ejecución #" + executionCounter);
            writer.println("S = " + states);
            writer.println("S0 = " + startState);
            writer.println("T = " + acceptStates);
            writer.println("A = " + alphabet);
            afdTransitions.forEach((key, value) -> writer.println("F(" + key + ") = {" + value + "}"));
            System.out.println("AFD guardado en " + filename);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    // Mostrar historial de cadenas ingresadas
    public void showValidationHistory() {
        if (validationHistory.isEmpty()) {
            System.out.println("No hay cadenas ingresadas.");
        } else {
            System.out.println("Historial de cadenas ingresadas:");
            for (String record : validationHistory) {
                System.out.println(record);
            }
        }
    }

    // Mostrar el AFN actual
    public void showAFN() {
        System.out.println("Estados: " + states);
        System.out.println("Estado inicial: " + startState);
        System.out.println("Estados de aceptación: " + acceptStates);
        System.out.println("Transiciones:");
        afnTransitions.forEach((key, value) -> System.out.println("F(" + key + ") = " + value));
    }

    // Método para generar la imagen del AFN
    public void generateAFNImage(String filename) {
        int imageWidth = 600;
        int imageHeight = 400;
        BufferedImage image = new BufferedImage(imageWidth, imageHeight, BufferedImage.TYPE_INT_ARGB);
        Graphics2D g = image.createGraphics();

        // Configuración básica de dibujo
        g.setRenderingHint(RenderingHints.KEY_ANTIALIASING, RenderingHints.VALUE_ANTIALIAS_ON);
        g.setColor(Color.WHITE);
        g.fillRect(0, 0, imageWidth, imageHeight);
        g.setColor(Color.BLACK);

        // Dibujar estados como círculos
        int stateRadius = 30;
        Map<Integer, Point> statePositions = new HashMap<>();
        int x = 100, y = 100;

        // Asegurarse de que el estado inicial se dibuje
        states.add(startState);

        for (int state : states) {
            statePositions.put(state, new Point(x, y));
            g.drawOval(x - stateRadius / 2, y - stateRadius / 2, stateRadius, stateRadius);
            g.drawString("q" + state, x - 10, y + 5);
            x += 100;
            if (x > imageWidth - 100) {
                x = 100;
                y += 100;
            }
        }

        // Dibujar transiciones como flechas
        for (Map.Entry<String, Set<Integer>> entry : afnTransitions.entrySet()) {
            String[] parts = entry.getKey().split(",");
            int fromState = Integer.parseInt(parts[0]);
            char symbol = parts[1].charAt(0);

            Point fromPos = statePositions.get(fromState);  // Verificar que no sea null

            if (fromPos == null) {
                continue;  // Omitir si no hay posición
            }

            for (int toState : entry.getValue()) {
                Point toPos = statePositions.get(toState);  // Verificar que no sea null

                if (toPos == null) {
                    continue;  // Omitir si no hay posición
                }

                drawArrow(g, fromPos.x, fromPos.y, toPos.x, toPos.y, symbol);
            }
        }

        // Guardar imagen en archivo
        try {
            ImageIO.write(image, "PNG", new File(filename));
            System.out.println("Imagen del AFN generada en " + filename);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    // Método para dibujar una flecha con un símbolo
    private void drawArrow(Graphics2D g, int x1, int y1, int x2, int y2, char symbol) {
        g.drawLine(x1, y1, x2, y2);
        g.drawString(String.valueOf(symbol), (x1 + x2) / 2, (y1 + y2) / 2);
    }

    public static void main(String[] args) {
        Proyecto automata = new Proyecto();
        Scanner scanner = new Scanner(System.in);

        while (true) {
            System.out.println("\n--- Menú ---");
            System.out.println("1. Ingreso de Expresión Regular");
            System.out.println("2. Generar AFN");
            System.out.println("3. Conversión a AFD");
            System.out.println("4. Validar Cadenas");
            System.out.println("5. Guardar AFD");
            System.out.println("6. Salir");
            System.out.println("7. Ver historial de cadenas ingresadas");
            System.out.println("8. Ver AFN actual");
            System.out.println("9. Generar imagen del AFN");
            System.out.print("Selecciona una opción: ");
            String option = scanner.nextLine();

            switch (option) {
                case "1":
                    System.out.print("Ingresa la expresión regular: ");
                    String expression = scanner.nextLine();
                    automata.generateAFNFromRegex(expression);
                    break;
                case "2":
                    automata.convertToAFD();
                    break;
                case "3":
                    automata.convertToAFD();
                    break;
                case "4":
                    System.out.print("Ingresa una cadena para validar (o '$' para salir): ");
                    String inputString = scanner.nextLine();
                    while (!inputString.equals("$")) {
                        boolean result = automata.validateString(inputString);
                        String resultString = inputString + " -> " + (result ? "No aceptada" : "Aceptada");
                        automata.validationHistory.add(resultString); // Guardar en historial
                        System.out.println(resultString);
                        System.out.print("Ingresa otra cadena para validar (o '$' para salir): ");
                        inputString = scanner.nextLine();
                    }
                    break;
                case "5":
                    automata.saveToFile("AFD.txt");
                    break;
                case "6":
                    System.out.println("Saliendo del programa.");
                    scanner.close();
                    return;
                case "7":
                    automata.showValidationHistory();
                    break;
                case "8":
                    automata.showAFN();
                    break;
                case "9":
                    System.out.print("Ingresa el nombre del archivo para guardar la imagen (ej. 'AFN.png'): ");
                    String imageName = scanner.nextLine();
                    automata.generateAFNImage(imageName);
                    break;
                default:
                    System.out.println("Opción no válida.");
            }
        }
    }
}
